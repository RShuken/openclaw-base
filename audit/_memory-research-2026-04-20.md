# Memory Extension Research — 2026-04-20

Scope: companion research for `skills/deploy-memory.md`. Audience already knows OpenClaw's built-in `sqlite-vec` + `workspace/memory/` indexing and wants to decide when/how to extend or replace it.

## Source caveats

- All specs/prices below are pulled from vendor docs, Ollama registry pages, or the project READMEs on 2026-04-20. Where a fact is second-hand (a blog post describing a GitHub issue, say) it's flagged inline.
- MTEB scores are not reproduced unless the source page quoted them — benchmark-claim drift is high for embedding models.
- Version numbers age fast; re-verify before quoting.
- I could not load every Reddit thread referenced; where I rely on an aggregator summary I say so.

---

## 1. Lossless Claw

- **URL:** https://github.com/Martian-Engineering/lossless-claw
- **Latest version / maintenance:** v0.9.2, released 2026-04-19. Actively maintained. 4.4k stars. MIT license.
- **Purpose:** Replace OpenClaw's default "chop and forget" context compaction (sliding-window truncation + lossy summarization) with lossless context management (LCM). Every message is kept in a SQLite store; older chunks are summarized into a DAG of summary nodes and agents retrieve detail on demand via dedicated tools.
- **Architecture:** SQLite for immutable message store. DAG of LLM-generated summary nodes rebuilt during compaction. Exposed to the agent through three tools: `lcm_grep` (keyword search across full history), `lcm_describe` (get summary of a time range / topic), `lcm_expand` (recover the original message text behind a summary).
- **Integration with OpenClaw:** Not a skill, not a side process — it hooks OpenClaw's pluggable **context engine slot**. OpenClaw 2026.4.x added this slot specifically so LCM can swap itself in. Config key in `openclaw.json`:
  ```json
  { "plugins": { "slots": { "contextEngine": "lossless-claw" } } }
  ```
- **Install (verbatim from README):**
  - `openclaw plugins install @martian-engineering/lossless-claw`
  - Local dev: `cd /path/to/lossless-claw && pnpm build && openclaw plugins install --link /path/to/lossless-claw`
- **Trade-offs:**
  - + Agent "never forgets" — compacted history is still recoverable on demand.
  - − Added latency: every compaction round costs LLM calls to build/update summary nodes.
  - − Storage grows monotonically (SQLite db per workspace — plan retention).
  - − Still needs an external memory layer for cross-workspace / cross-agent recall; LCM is a per-session context engine, not a global memory.
- **Community reception:** 4.4k stars in ~6 months is strong. Josh Lehman (x.com/jlehman_, post id 2033319118343184725) publicly differentiates it from "memory systems": LCM handles in-context retention, mem0/LanceDB handle external recall — use both together.
- **Citations:** https://github.com/Martian-Engineering/lossless-claw ; https://x.com/jlehman_/status/2033319118343184725

---

## 2. mem0 ("Mem0" / "memzero")

- **URL:** https://github.com/mem0ai/mem0 — 53.6k stars. Apache 2.0. Current line: v1.0.x (stable) + v2.0.0b0 Python SDK pre-release.
- **Purpose:** Universal memory layer for AI agents. Extracts entities and salient facts from conversation, stores them across a multi-level hierarchy (user / session / agent), and injects relevant memories back into prompts.
- **Architecture:** Hybrid. Default embedding = OpenAI `text-embedding-3-small`. Vector store is pluggable (Qdrant, Chroma, pgvector, LanceDB…). Retrieval fuses three signals: semantic vector + BM25 keyword + entity-linking graph. Benchmark claims from the README: 91.6 on LoCoMo, 93.4 on LongMemEval, 64.1 on BEAM-1M (these are Mem0's own numbers — verify independently before citing externally).
- **OpenClaw integration:** Multiple mem0-backed OpenClaw plugins exist — the ecosystem is fragmented but real:
  - **Official path:** https://docs.mem0.ai/integrations/openclaw and https://github.com/mem0ai/mem0/tree/main/openclaw (canonical integration, maintained by mem0 team).
  - **Community forks:** `serenichron/openclaw-memory-mem0`, `tensakulabs/openclaw-mem0`, `abhpc/openclaw-mem0`, `xRay2016/openclaw-mem0-plugin`. These are mostly the same pattern: replace OpenClaw's LanceDB memory backend with calls to a self-hosted Mem0 REST API. Any OpenAI-compatible provider works for fact extraction and embeddings.
  - The plugin exposes three tools — `memory_recall`, `memory_store`, `memory_forget` — plus **auto-recall** (inject relevant memories before every agent run) and **auto-capture** (summarize and store after every run). Session memory is scoped via Mem0's `run_id`; user memory persists across sessions.
- **Install (self-hosted, verbatim):**
  - Python: `pip install mem0ai` (or `pip install mem0ai[nlp]` + `python -m spacy download en_core_web_sm` for NLP entity extraction).
  - Node: `npm install mem0ai`.
  - OpenClaw plugin (community pattern): copy plugin to `~/.openclaw/extensions/memory-mem0`, run `npm install`, then configure `openclaw.json` with `{ baseUrl, userId, autoCapture, autoRecall }`.
- **Cloud vs self-hosted:** Hosted service at app.mem0.ai (managed, analytics, auto-updates). Self-host via the Apache-2.0 repo — you supply the LLM, embedding model, and vector store. Pricing for the hosted tier isn't published on the public README (need dashboard login).
- **Trade-offs vs OpenClaw built-in:**
  - + Structured semantic memory (facts, entities, relationships), not just opaque vector chunks.
  - + Built-in cross-session scoping (user vs session memory).
  - − Extra moving part: another REST service + vector DB to run/monitor.
  - − Fact extraction = extra LLM calls on every conversation turn (cost).
  - − Community plugin variants aren't all battle-tested; pick the official `mem0ai/mem0/openclaw` path unless you know why you want a fork.
- **Citations:** https://github.com/mem0ai/mem0 ; https://docs.mem0.ai/integrations/openclaw ; https://mem0.ai/blog/add-persistent-memory-openclaw ; https://arxiv.org/abs/2504.19413

---

## 3. Local embedding models via Ollama

Pulled from https://ollama.com/search?c=embedding on 2026-04-20. Ranked roughly by popularity (pulls) and use-case fit.

| Model | Params | Ollama pulls | Size on disk | Dims | Context | License | Notes |
|---|---|---|---|---|---|---|---|
| `nomic-embed-text` | ~137M | 65.8M | ~300 MB | 768 | 8192 | Apache 2.0 | Baseline workhorse. Beats OpenAI `ada-002` and `text-embedding-3-small` on short+long per Ollama blog. First choice for "just make it work locally." |
| `mxbai-embed-large` | 335M | 9.7M | ~700 MB | 1024 | 512 | Apache 2.0 | MixedBread. Strong MTEB on BERT-large class; short context is its main limitation. |
| `bge-m3` | 567M | 3.9M | ~1.2 GB | 1024 | 8192 | MIT | BAAI. 100+ languages. Only Ollama embedding model that supports dense + sparse + ColBERT multi-vector retrieval simultaneously — valuable if you want hybrid in a single model. |
| `snowflake-arctic-embed` family | 22M → 335M | 3M | varies | varies | — | Apache 2.0 | Multiple size tags (22m/33m/110m/137m/335m). Use `:s` or `:xs` for tiny machines; `:l` for quality. |
| `snowflake-arctic-embed2` | ~568M | 370k | 1.2 GB | not published on Ollama page (verify before sizing DB) | 8K | Apache 2.0 | Multilingual, Matryoshka (compress to 128-byte vectors), sub-10ms query latency on A10. |
| `embeddinggemma` | 308M | 1M | <200 MB quantized | 768 default, MRL down to 128 | 2048 | Gemma terms | See §4 — this is the actual "Gemma for embeddings." |
| `qwen3-embedding` | 0.6B / 4B / 8B | 1.8M | varies | varies (0.6b commonly used with 1024 dim) | — | Apache 2.0 | Qwen 3 line. 0.6b is a sensible CPU choice; 7b/8b is the top of the open benchmark when you have GPU. Mem0 README explicitly recommends `Qwen 600M or comparable` for hybrid search. |
| `granite-embedding` | 30M / 278M | 317k | small | 384 / 768 | — | Apache 2.0 | IBM Granite. Business-oriented, multilingual. |
| `all-minilm` | 22M / 33M | 3M | ~50 MB | 384 | 512 | Apache 2.0 | Tiny. Use for hard-constrained RAM (edge device, Raspberry Pi class). Quality tier: low. |
| `bge-large` | 335M | 258k | ~700 MB | 1024 | 512 | MIT | Older BAAI baseline. Superseded by bge-m3 for most cases. |
| `paraphrase-multilingual` | 278M | 826k | — | 768 | — | Apache 2.0 | Sentence-transformers port. Multilingual but older than bge-m3. |
| `nomic-embed-text-v2-moe` | — | 182k | — | — | — | Apache 2.0 | Nomic's MoE successor. Not enough adoption yet to recommend over v1 by default. |

**Quality tier summary (qualitative, from vendor+benchmark pages):**
- Top: `qwen3-embedding:8b`, `snowflake-arctic-embed2`, `bge-m3`.
- Middle: `mxbai-embed-large`, `nomic-embed-text`, `embeddinggemma`.
- Budget / edge: `all-minilm`, `snowflake-arctic-embed:s`, `granite-embedding:30m`.

Citations: https://ollama.com/search?c=embedding ; https://ollama.com/blog/embedding-models ; https://ollama.com/library/nomic-embed-text ; https://ollama.com/library/mxbai-embed-large ; https://ollama.com/library/snowflake-arctic-embed2 ; https://ollama.com/library/embeddinggemma ; https://huggingface.co/Snowflake/snowflake-arctic-embed-l-v2.0

---

## 4. Gemma family for embeddings

Short answer: **yes — use `EmbeddingGemma` (Google's dedicated Gemma-based embedding model), not base Gemma 2 / Gemma 3 chat weights.**

- `EmbeddingGemma` (a.k.a. `google/embeddinggemma-300m`, `ollama pull embeddinggemma`) is 308M parameters, built on the Gemma 3 transformer backbone but **converted to bi-directional attention** (encoder, not decoder). So it's a legit embedding model — not a chat model repurposed with mean pooling.
- Output dimensions: 768 default, Matryoshka-reducible to 512 / 256 / 128.
- Context: 2K tokens.
- Footprint: <200 MB RAM quantized, <22 ms on EdgeTPU. Designed for on-device.
- MTEB: highest-ranking open multilingual embedding model under 500M params (per Google's own claim — verify).
- License: Gemma license terms (permissive for most use but read it; it is **not** Apache/MIT).
- **The user may be conflating with Gemini embeddings** — see §5. `gemini-embedding-001` is the cloud API; `embeddinggemma` is the local sibling. They are different models, different licenses, different quality tiers.
- Do **not** try to roll your own embedding by mean-pooling Gemma 2 / 3 chat logits. That works technically, but quality is well below EmbeddingGemma and you'd be reinventing wheel + eating the license risk.

Citations: https://ai.google.dev/gemma/docs/embeddinggemma ; https://developers.googleblog.com/en/introducing-embeddinggemma/ ; https://huggingface.co/google/embeddinggemma-300m ; https://arxiv.org/abs/2509.20354 ; https://ollama.com/library/embeddinggemma

---

## 5. Gemini cloud embeddings

- **Current default in deploy-memory.md:** `gemini-embedding-001`.
- **Dimensions:** 3072 default (Matryoshka). Officially supported reductions: **768, 1536, 3072**. All three are the recommended sizes; don't pick arbitrary values.
- **Per-request limits:** up to 250 input texts per request; 20,000 tokens per request; **only the first 2,048 tokens per input text are used** (hard truncation — relevant for long `memory/YYYY-MM-DD.md` files).
- **Pricing:** $0.15 per 1M input tokens standard; $0.075 per 1M batch. (Source: Google Developers blog, Apr 2026.)
- **Rate limits:** quota is **tokens-per-minute per project**, not RPM. Free-tier access is via AI Studio; exact free-tier TPM isn't published on a stable URL — check `ai.google.dev/gemini-api/docs/rate-limits` for the current number. Free tier **does** include embeddings but is the first thing to exhaust on a 24/7 agent.
- **Newer models:** As of 2026-04-20 I did not find a shipped `gemini-embedding-002` in the public Gemini API docs. There's a "Gemini Embedding 2" being discussed in external blog posts (e.g. mayhemcode.com, March 2026) framed as multimodal (text/image/video/audio). This is **not confirmed in Google docs** — treat as unreleased-or-preview until `ai.google.dev/gemini-api/docs/embeddings` lists it.

Citations: https://developers.googleblog.com/gemini-embedding-available-gemini-api/ ; https://ai.google.dev/gemini-api/docs/embeddings ; https://ai.google.dev/gemini-api/docs/pricing ; https://ai.google.dev/gemini-api/docs/rate-limits

---

## 6. OpenAI cloud embeddings

- `text-embedding-3-small` — 1536 dims default (Matryoshka-truncatable via `dimensions` param), **$0.02 per 1M tokens** standard / **$0.01 per 1M** batch.
- `text-embedding-3-large` — 3072 dims default (Matryoshka-truncatable), **$0.13 per 1M tokens** standard / **$0.065 per 1M** batch.
- Both: trained with Matryoshka so you can shorten embeddings losslessly via the `dimensions` parameter.
- **Newer:** I did not find a `text-embedding-4` in OpenAI docs as of 2026-04-20. The public models page for embeddings still lists only the `-3-small`/`-3-large` pair. Any "embedding-4" reference you see elsewhere is unconfirmed.

Citations: https://platform.openai.com/docs/guides/embeddings ; https://developers.openai.com/api/docs/models/text-embedding-3-small ; https://developers.openai.com/api/docs/models/text-embedding-3-large ; https://openai.com/index/new-embedding-models-and-api-updates/

---

## 7. Community-reported OpenClaw memory issues (recap + new finds)

Confirmed from `audit/_baseline.md §K` plus fresh 2026 citations:

- **Compaction → silent hallucinations.** Multi-step procedures and structured data get flattened into summaries that the agent later treats as ground truth. Baseline §K.339. The "Mind Your HEARTBEAT!" paper (arxiv 2603.23064v2) adds an adversarial angle: polluted content in background heartbeat sessions can persist across compaction with attack success rates up to 75.6%. (This is an attack paper, not just a reliability bug — a memory design needs to consider it if heartbeat processes the open web.)
- **`heartbeat.model` / `compaction.model` override broken** — https://github.com/openclaw/openclaw/issues/58137 (closed — created 2026-03-31). The live-session model-switch detector throws `LiveSessionModelSwitchError` and forces compaction/heartbeat back to the expensive primary model. Related follow-up: https://github.com/openclaw/openclaw/issues/65218 (2026-04-11 — precheck ignores `compaction reserve`, forces `reserveTokens=16384`). Bottom line: the local-routing advice in `feedback_openclaw_local_background_routing.md` won't take effect on affected versions — test it after deploy.
- **Memory folder ≥400 files → compaction collapses quality.** Baseline §K.352. Community fix: weekly rollup + hierarchical folders (see §9).
- **Dreaming pollutes daily files** — https://github.com/openclaw/openclaw/issues/66947. The dreaming subsystem writes `memory/YYYY-MM-DD.md` and then heartbeat sees the file as non-empty and skips logging real activity.
- **Pre-compaction memory-flush bug** — https://github.com/openclaw/openclaw/issues/44787. Agent overwrites existing daily memory files instead of appending.
- **Aggregator commentary:**
  - Daily Dose of DS (Feb 2026) — argues OpenClaw memory can't reason about relationships, no provenance, cross-project bleed. Recommends a knowledge-graph plugin (Cognee). https://blog.dailydoseofds.com/p/openclaws-memory-is-broken-heres
  - VelvetShark "Memory Masterclass" — advocates hierarchical flat-kernel layout with daily `memory/YYYY-MM-DD.md` + weekly promotion into `MEMORY.md`. Doesn't address compaction-induced hallucinations directly; frames compaction as "lossy summarization" rather than "invents false claims." https://velvetshark.com/openclaw-memory-masterclass
  - kaxo.io "8 Silent Failures" — general production gotchas. https://kaxo.io/insights/openclaw-production-gotchas/

---

## 8. Other vector DB / memory systems

- **LanceDB** — https://github.com/lancedb/lancedb. Embedded-first, columnar, disk-based. Ideal when the index is larger than RAM. OpenClaw has a `memory-lancedb` cloud-storage variant mentioned in baseline §K.231 and used by `OpenClaw_MemoryFix` (https://github.com/openclaw042375-OpenClaw/OpenClaw_MemoryFix — experimental, 0 stars, 1 commit; treat as demo not production).
- **Chroma** — https://github.com/chroma-core/chroma. Simplest API, lightweight, SQLite-backed, single-VPS-friendly (4–8 GB RAM handles millions of embeddings per vendor blog). Best "zero to prototype" pick.
- **Qdrant** — https://github.com/qdrant/qdrant. Rust, self-hostable, strong metadata filtering + dynamic sharding. Pick when you need rich filter predicates (e.g., "memories tagged project=X, date>Y, confidence>0.7").
- **Weaviate** — https://github.com/weaviate/weaviate. Self-hostable, built-in vectorization modules (you can insert raw text and it embeds server-side), supports multi-modal (text/image). Heavier operational footprint.
- **Letta (formerly MemGPT)** — https://github.com/letta-ai/letta. Self-editing memory agent framework. Originated the "agent rewrites its own core memory blocks" pattern. Now also ships `letta-code` — a memory-first coding harness portable across Claude/GPT/Gemini/GLM/Kimi. **Viability with OpenClaw:** no native OpenClaw plugin as of Apr 2026. You could call Letta as an MCP server from OpenClaw, but you'd be running two agent frameworks side-by-side. Cleaner as a replacement than as an extension.
- **Zep** — https://github.com/getzep/zep (service) + https://github.com/getzep/graphiti (core, open source). Temporal knowledge graph — every fact has a validity window ("Kendra loves Adidas shoes, as of 2026-03-14; superseded 2026-04-02"). Hybrid retrieval = semantic + BM25 + graph traversal; claims sub-200ms latency. Outperforms MemGPT on DMR benchmark (per their arxiv paper 2501.13956 — vendor-reported, verify). No first-party OpenClaw plugin yet; integration would be via HTTP/MCP.
- **Cognee** — mentioned in daily-dose-of-ds post; open-source knowledge-graph memory engine with a documented OpenClaw plugin path. Less battle-tested than the above.
- **MemOS Cloud OpenClaw Plugin** — https://github.com/MemTensor/MemOS-Cloud-OpenClaw-Plugin. Official MemOS Cloud plugin with auto-recall before / auto-save after each agent run. Commercial service behind it.
- **OpenMemory** — https://github.com/CaviraOSS/OpenMemory. Local persistent memory store targeting Claude Desktop, Copilot, Codex, Antigravity. Not OpenClaw-specific but model-agnostic.

---

## 9. Community patterns for OpenClaw memory

All from either `audit/_baseline.md §K` or the articles cited above.

- **Weekly rollup** — every Monday compact the last week's `memory/YYYY-MM-DD.md` into a single summary file, archive originals. Prevents the `>400 files → compaction quality collapse` failure mode. (velvetshark, baseline §K.352/.377).
- **Hierarchical folders inside `memory/`** — split by project/entity/topic instead of one flat date-named pile. Trades file-path discipline for survivable scale. (baseline §K.352).
- **Kernel vs memory split** — flat-root `.md` files (AGENTS/SOUL/IDENTITY/HEARTBEAT/MEMORY/TOOLS/USER/BOOTSTRAP/BOOT) are always loaded; `memory/YYYY-MM-DD.md` are vector-indexed and retrieved on demand. Keep the kernel under the stable token budget. (baseline §K.389).
- **MEMORY.md security gate** — never load MEMORY.md in group chats or sub-agents; boot sequence must enforce. Baseline §§K.153, K.405, K.430. Applies regardless of which memory backend you pick.
- **Identity split** — SOUL.md (philosophy) + IDENTITY.md (name/voice) + MEMORY.md (index). Baseline §K.351.
- **Entity extraction via plugin** — handled by mem0 (§2), Zep/Graphiti (§8), or Cognee. Community pattern is: don't try to entity-extract in pure markdown; delegate to a structured store.
- **Hybrid model routing for background writes** — primary (cloud) model only for user-facing chats; heartbeat / dreaming / embeddings run on local Ollama. Matches `feedback_openclaw_local_background_routing.md`. Blocked on the issue-58137/65218 bugs — verify on target OpenClaw version.

---

## 10. Decision tree: when to pick what

**Starting position** (deploy-memory.md default): OpenClaw built-in memory + `sqlite-vec` + Gemini `gemini-embedding-001` (3072 dim) via `auth-profiles.json`. Works for a single agent, single-user, under ~a few hundred memory files.

- **Zero outbound network required** → swap `gemini-embedding-001` for `embeddinggemma` or `nomic-embed-text` via Ollama. Keep OpenClaw's built-in sqlite-vec backend. Storage cost: negligible. Quality cost: modest.
- **Best raw retrieval quality, cost insensitive** → `gemini-embedding-001` at 3072 dim or `text-embedding-3-large` at 3072 dim. Difference between the two is small; pick by which key you already have.
- **Cost-sensitive but still cloud** → `text-embedding-3-small` at 1536 dim ($0.02/M tokens) — cheapest reliable cloud option. Mem0's default for a reason.
- **Hit Gemini free quota on a 24/7 agent** → move to `text-embedding-3-small` (pennies/month) **or** go local with `nomic-embed-text`. Do not try to stretch the free tier on a heartbeat-driven agent; embedding calls per heartbeat will eat it.
- **Privacy / air-gap hard requirement** → Ollama embedding + local vector DB (LanceDB or Chroma). No third-party network. Pair with local LLM so the whole loop is offline.
- **24/7 always-on, minimize $** → Ollama (`nomic-embed-text` or `embeddinggemma`) for embeddings, cloud frontier model only for user-chat turns. Matches `feedback_openclaw_local_background_routing.md`. Warning: the heartbeat.model bugs (§7) may undo this until patched — verify on target version.
- **Agent is hallucinating compacted facts as real** → install **Lossless Claw** in the `contextEngine` slot. This is the direct fix for the §7 compaction-hallucination mode; keeps full history recoverable.
- **Want structured semantic memory (entities, relationships, provenance), not opaque vector chunks** → **mem0** (simplest, largest community) or **Zep/Graphiti** (when relationships have time validity) or **Letta** (when you want self-editing memory blocks as a first-class pattern). Pick one; don't stack three entity systems.
- **Need cross-session recall across many agents/users** → mem0's user/session scoping is the cleanest fit. Zep works too but costs more to operate.
- **Index is larger than RAM** → LanceDB. Chroma and sqlite-vec start hurting past tens of millions of chunks; LanceDB's disk-first design handles it.
- **Need complex filter predicates ("memories where project=X AND confidence>0.7 AND date>...")** → Qdrant.
- **Hybrid search in one model** (dense + sparse + ColBERT) → `bge-m3` as the embedding model; retrieve with all three signals from the same vectors.

**Stacking recommendation (most deployments, April 2026):**
1. Lossless Claw in the context engine slot (fixes compaction hallucination).
2. OpenClaw built-in `sqlite-vec` for the workspace memory folder (fast, local, works).
3. `text-embedding-3-small` (cheap cloud) or `embeddinggemma` (local) for the embedding provider.
4. Add mem0 *only if* structured entity memory is a real requirement — skip it otherwise; the extra service isn't free to operate.

---

## 11. Citations

- https://github.com/Martian-Engineering/lossless-claw
- https://x.com/jlehman_/status/2033319118343184725
- https://github.com/mem0ai/mem0
- https://github.com/mem0ai/mem0/tree/main/openclaw
- https://docs.mem0.ai/integrations/openclaw
- https://mem0.ai/blog/add-persistent-memory-openclaw
- https://arxiv.org/abs/2504.19413
- https://github.com/serenichron/openclaw-memory-mem0
- https://github.com/tensakulabs/openclaw-mem0
- https://github.com/MemTensor/MemOS-Cloud-OpenClaw-Plugin
- https://github.com/CaviraOSS/OpenMemory
- https://ollama.com/search?c=embedding
- https://ollama.com/blog/embedding-models
- https://ollama.com/library/nomic-embed-text
- https://ollama.com/library/mxbai-embed-large
- https://ollama.com/library/bge-m3
- https://ollama.com/library/snowflake-arctic-embed2
- https://ollama.com/library/embeddinggemma
- https://ai.google.dev/gemma/docs/embeddinggemma
- https://developers.googleblog.com/en/introducing-embeddinggemma/
- https://huggingface.co/google/embeddinggemma-300m
- https://arxiv.org/abs/2509.20354
- https://developers.googleblog.com/gemini-embedding-available-gemini-api/
- https://ai.google.dev/gemini-api/docs/embeddings
- https://ai.google.dev/gemini-api/docs/pricing
- https://ai.google.dev/gemini-api/docs/rate-limits
- https://platform.openai.com/docs/guides/embeddings
- https://developers.openai.com/api/docs/models/text-embedding-3-small
- https://developers.openai.com/api/docs/models/text-embedding-3-large
- https://openai.com/index/new-embedding-models-and-api-updates/
- https://github.com/openclaw/openclaw/issues/58137
- https://github.com/openclaw/openclaw/issues/65218
- https://github.com/openclaw/openclaw/issues/66947
- https://github.com/openclaw/openclaw/issues/44787
- https://github.com/openclaw/openclaw/blob/main/docs/concepts/memory.md
- https://arxiv.org/html/2603.23064v2
- https://blog.dailydoseofds.com/p/openclaws-memory-is-broken-heres
- https://velvetshark.com/openclaw-memory-masterclass
- https://kaxo.io/insights/openclaw-production-gotchas/
- https://github.com/lancedb/lancedb
- https://github.com/chroma-core/chroma
- https://github.com/qdrant/qdrant
- https://github.com/weaviate/weaviate
- https://github.com/letta-ai/letta
- https://github.com/letta-ai/letta-code
- https://www.letta.com/blog/memgpt-and-letta
- https://github.com/getzep/zep
- https://github.com/getzep/graphiti
- https://arxiv.org/abs/2501.13956
- https://github.com/openclaw042375-OpenClaw/OpenClaw_MemoryFix
- https://github.com/alvinreal/awesome-openclaw
- audit/_baseline.md §K
