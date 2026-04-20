# Memory options — index

This directory holds one file per memory-architecture option. Each file has YAML frontmatter with the fields an audit agent needs to verify it (repo URL, license, version, last-checked date, maintenance status).

**Parent skill:** [`../deploy-memory-advanced.md`](../deploy-memory-advanced.md) — the routing document that points here.

**Template for adding new options:** [`_template.md`](./_template.md)

## Index

| File | Name | Category | Status | Repo | Last checked |
|------|------|----------|--------|------|--------------|
| [`lossless-claw.md`](./lossless-claw.md) | Lossless Claw | context-engine | active (v0.9.2) | [github.com/Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw) | 2026-04-20 |
| [`mem0.md`](./mem0.md) | mem0 | structured-memory | active (v1.0.x) | [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0) | 2026-04-20 |
| [`ollama-embeddings.md`](./ollama-embeddings.md) | Ollama local embeddings | embedding-local | active | [ollama.com/search?c=embedding](https://ollama.com/search?c=embedding) | 2026-04-20 |
| [`omlx-apple-silicon.md`](./omlx-apple-silicon.md) | oMLX (Apple Silicon) | embedding-local + chat-heartbeat | active | [github.com/jundot/omlx](https://github.com/jundot/omlx) | 2026-04-20 |
| [`embeddinggemma.md`](./embeddinggemma.md) | EmbeddingGemma | embedding-local | active | [ai.google.dev/gemma/docs/embeddinggemma](https://ai.google.dev/gemma/docs/embeddinggemma) | 2026-04-20 |
| [`gemini-embeddings.md`](./gemini-embeddings.md) | Gemini cloud embeddings | embedding-cloud | active (v1 stable + v2 preview) | [ai.google.dev/gemini-api/docs/embeddings](https://ai.google.dev/gemini-api/docs/embeddings) | 2026-04-20 |
| [`openai-embeddings.md`](./openai-embeddings.md) | OpenAI cloud embeddings | embedding-cloud | active | [platform.openai.com/docs/guides/embeddings](https://platform.openai.com/docs/guides/embeddings) | 2026-04-20 |
| [`graphiti-neo4j.md`](./graphiti-neo4j.md) | Graphiti + Neo4j (self-hosted) | structured-memory | active | [github.com/getzep/graphiti](https://github.com/getzep/graphiti) | 2026-04-20 |
| [`gemma4-heartbeat.md`](./gemma4-heartbeat.md) | Gemma 4 (local heartbeat model) | chat-heartbeat | active (released 2026-04-02) | [deepmind.google/models/gemma/gemma-4](https://deepmind.google/models/gemma/gemma-4/) | 2026-04-20 |
| [`vector-dbs-alternatives.md`](./vector-dbs-alternatives.md) | LanceDB / Chroma / Qdrant / Weaviate | vector-db | active | multiple | 2026-04-20 |
| [`letta-memgpt.md`](./letta-memgpt.md) | Letta (MemGPT) | structured-memory | active (no native OpenClaw plugin) | [github.com/letta-ai/letta](https://github.com/letta-ai/letta) | 2026-04-20 |
| [`zep-hosted.md`](./zep-hosted.md) | Zep (hosted temporal KG) | structured-memory | active (service) | [github.com/getzep/zep](https://github.com/getzep/zep) | 2026-04-20 |
| [`community-patterns.md`](./community-patterns.md) | Community patterns (Hot/Warm/Cold, weekly rollup, etc.) | community-pattern | active | baseline §K | 2026-04-20 |
| [`enterprise-v2-stack-template.md`](./enterprise-v2-stack-template.md) | {{AGENT_NAME}} Stack — full reference template | template | active (reference-snapshot deployed 2026-03-31) | composition | 2026-04-20 |

## Categories

- **context-engine** — plugs into OpenClaw's `contextEngine` slot (Lossless Claw)
- **structured-memory** — entity/fact/relationship storage (mem0, Graphiti+Neo4j, Letta, Zep)
- **embedding-local** — run embeddings on the local machine (Ollama, oMLX, EmbeddingGemma)
- **embedding-cloud** — managed cloud embedding APIs (Gemini, OpenAI)
- **vector-db** — alternative vector stores to sqlite-vec (LanceDB, Chroma, Qdrant, Weaviate)
- **chat-heartbeat** — local chat/reasoning/tool-use models for heartbeats + entity extraction (Gemma 4)
- **community-pattern** — additive patterns that apply to any backend
- **template** — pre-composed stacks for common deployment shapes ({{AGENT_NAME}} Stack)

## How to add a new option

1. Copy [`_template.md`](./_template.md) → `<new-option>.md`
2. Fill in the frontmatter YAML fields
3. Write the body (Purpose / When to pick / Install / Verification / Trade-offs / Citations)
4. Add a row to the Index table in this README
5. Add an entry to the parent routing doc ([`../deploy-memory-advanced.md`](../deploy-memory-advanced.md)) Options Matrix
6. If it maps to a stack template, update the relevant template file

## For audit / update-checker agents

Every option file has YAML frontmatter at the top with these machine-parseable fields:

- `name`, `category`, `repo`, `license`
- `version_last_verified`, `last_checked` (bump when you re-check)
- `maintenance_status` (`active` / `dormant` / `deprecated` / `preview` / `unverified`)
- `openclaw_integration`, `cost_tier`, `privacy_tier`
- `requires` (array), `docs_urls` (array)

An audit agent should:
1. Parse every `memory-options/*.md` frontmatter
2. Fetch the `repo` URL + `docs_urls` — check for new releases / issues / deprecations
3. Update `version_last_verified` + `last_checked` in the frontmatter
4. Flag any file whose `maintenance_status` has changed (e.g., repo archived, maintainer walked away)
5. Report a summary PR of what moved

No single file should need to be re-read to audit another. Each option is independently verifiable.
