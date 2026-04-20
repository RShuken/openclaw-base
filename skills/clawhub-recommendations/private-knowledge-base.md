---
name: "private-knowledge-base"
clawhub_slug: "@-/private-knowledge-base"
clawhub_url: "https://clawhub.ai/@-/private-knowledge-base"
category: "research"
author: "<VERIFY — baseline §B lists bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-knowledge-base.md (evaluate; overlaps bundled `openclaw memory`)"
cost_tier: "api-key-required"
privacy_tier: "hybrid"
docs_urls:
  - "https://clawhub.ai/@-/private-knowledge-base"
---

# private-knowledge-base

## What it is

RAG-style knowledge-base skill — ingest URLs/documents, semantic search, inject results into agent prompts. Baseline §B lists `private-knowledge-base`, `rag-search`, `hk101-living-rag`, `sulada-knowledge-base` as strong matches for our `deploy-knowledge-base.md`.

## Why we recommend it

- OpenClaw 2026.4.15+ ships a built-in `memory` subsystem (sqlite-vec). The ClawHub KB skills add ingestion (URL / YouTube / PDF / X) on top — our `deploy-knowledge-base.md` does the same thing.
- Install-and-go vs. operating the ingestion pipeline ourselves.

## When to pick this over our own deploy-*

- Greenfield KB installs where the client's source list is well-modeled by the skill's connectors.
- Small/medium KBs (<10k docs) where the skill's default storage is fine.

## When NOT to pick this

- Large KBs (>100k docs) that need a dedicated vector DB (see `../memory-options/vector-dbs-alternatives.md`).
- Client already has a KB in Notion / Drive and wants it queried in place (use gog/notion search directly).

## Install

```bash
clawhub install private-knowledge-base
# Baseline lists four alternatives — pick based on source-connector coverage.
```

## Configure / wire into OpenClaw

Route embedding calls through a local embedder if privacy-sensitive — see `../memory-options/embeddinggemma.md` or `../memory-options/ollama-embeddings.md`. Route query-synthesis through a cheap model (`KB_QUERY_MODEL` env var per `../deploy-knowledge-base.md`).

## Known issues / gotchas

- Ingestion LLM calls balloon cost if routed through a frontier model — always cheap-model them.
- Overlaps with bundled `openclaw memory` — decide which is canonical for the install; don't run both unless deliberate.

## Citations

- https://clawhub.ai/@-/private-knowledge-base
- `skills/deploy-knowledge-base.md`
- `audit/_baseline.md` §B
