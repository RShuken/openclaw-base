---
name: "Zep (hosted temporal knowledge graph)"
category: "structured-memory"
repo: "https://github.com/getzep/zep"
license: "varies (service; Graphiti core is MIT)"
version_last_verified: "current service"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "HTTP-bridge (no native plugin)"
cost_tier: "hosted-service"
privacy_tier: "cloud"
requires:
  - "Zep account"
docs_urls:
  - "https://github.com/getzep/zep"
  - "https://github.com/getzep/graphiti"
  - "https://arxiv.org/abs/2501.13956"
---

# Zep — managed temporal knowledge graph

- **Service:** [getzep.com](https://www.getzep.com) — managed long-term memory for LLM apps
- **Open-source core:** [Graphiti](https://github.com/getzep/graphiti) (see `graphiti-neo4j.md` for the self-hosted variant)
- **Paper:** [arxiv 2501.13956](https://arxiv.org/abs/2501.13956) — outperforms MemGPT on DMR benchmark (vendor-reported, verify)

## What it is

Temporal knowledge graph service where every fact has a validity window ("X loves Adidas as of 2026-03-14; superseded 2026-04-02"). Hybrid retrieval = semantic + BM25 + graph traversal. Sub-200ms latency claimed.

## When to pick hosted Zep over self-hosted `graphiti-neo4j.md`

- Don't want to run Docker + Neo4j + Flask + Python + Node.js yourself
- OK with data leaving the machine (not a VC-firm-grade privacy requirement)
- Want vendor-supplied auto-ingestion and analytics
- Willing to pay for operational simplicity

## When to pick self-hosted Graphiti + Neo4j instead

- Privacy hard requirement (enterprise-grade: confidential deal data can't leave machine)
- Cost-sensitive long-term
- Want Cypher query access

## OpenClaw integration

No first-party plugin. Integration via HTTP API or MCP server.

## Citations

- https://github.com/getzep/zep
- https://github.com/getzep/graphiti
- https://arxiv.org/abs/2501.13956
