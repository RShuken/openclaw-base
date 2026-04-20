---
name: "Alternative vector DBs (LanceDB / Chroma / Qdrant / Weaviate)"
category: "vector-db"
repo: "multiple"
license: "varies (mostly permissive)"
version_last_verified: "registry snapshot 2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "plugin / service bridge (varies)"
cost_tier: "free-local"
privacy_tier: "fully-local"
requires:
  - "varies per DB"
docs_urls:
  - "https://github.com/lancedb/lancedb"
  - "https://github.com/chroma-core/chroma"
  - "https://github.com/qdrant/qdrant"
  - "https://github.com/weaviate/weaviate"
---

# Alternative vector DBs — replace sqlite-vec

For most personal deployments, **OpenClaw's built-in sqlite-vec is plenty**. Reach for these only when you hit a specific constraint.

## Decision table

| DB | Repo | When to pick |
|----|------|--------------|
| [LanceDB](https://github.com/lancedb/lancedb) | `lancedb/lancedb` | Index larger than RAM — disk-first columnar design handles tens of millions of chunks. OpenClaw has a `memory-lancedb` cloud-storage variant |
| [Chroma](https://github.com/chroma-core/chroma) | `chroma-core/chroma` | Simplest API. Lightweight SQLite-backed. Single VPS with 4-8 GB RAM handles millions of embeddings per vendor blog. Zero-to-prototype pick |
| [Qdrant](https://github.com/qdrant/qdrant) | `qdrant/qdrant` | Rust, self-hostable, strong metadata filtering + dynamic sharding. Pick when you need rich filters ("memories tagged project=X, date>Y, confidence>0.7") |
| [Weaviate](https://github.com/weaviate/weaviate) | `weaviate/weaviate` | Self-hostable, built-in server-side vectorization (insert raw text, it embeds for you), multimodal. Heavier operational footprint |

## Less mature (noted, verify before adopting)

- [`OpenClaw_MemoryFix`](https://github.com/openclaw042375-OpenClaw/OpenClaw_MemoryFix) — experimental, 0 stars, 1 commit; treat as demo
- [Cognee](https://blog.dailydoseofds.com/p/openclaws-memory-is-broken-heres) — open-source knowledge-graph memory engine with OpenClaw plugin path
- [MemOS Cloud Plugin](https://github.com/MemTensor/MemOS-Cloud-OpenClaw-Plugin) — official MemOS Cloud plugin; commercial service behind it
- [OpenMemory](https://github.com/CaviraOSS/OpenMemory) — local persistent memory store; not OpenClaw-specific, targets Claude Desktop / Copilot / Codex / Antigravity

## When NOT to bother

Unless one of the specific constraints above bites you, **sqlite-vec is the right answer**. Adding another service is operational overhead most deployments don't need. The built-in memory subsystem is genuinely good.

## Citations

- https://github.com/lancedb/lancedb
- https://github.com/chroma-core/chroma
- https://github.com/qdrant/qdrant
- https://github.com/weaviate/weaviate
