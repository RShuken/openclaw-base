---
name: "Graphiti + Neo4j (self-hosted temporal knowledge graph)"
category: "structured-memory"
repo: "https://github.com/getzep/graphiti"
license: "MIT (Graphiti); GPLv3 (Neo4j Community)"
version_last_verified: "Graphiti current main branch"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "HTTP-bridge + neo4j-driver (Node)"
cost_tier: "extra-llm-calls"
privacy_tier: "fully-local"
requires:
  - "Docker"
  - "Python 3.11+"
  - "Node.js 22+"
  - "16GB+ RAM recommended"
docs_urls:
  - "https://github.com/getzep/graphiti"
  - "https://neo4j.com/download/neo4j-community-edition/"
---

# Graphiti + Neo4j — self-hosted temporal knowledge graph

**This is the reference snapshot's {{AGENT_NAME}} choice**, elevated to Week-1 install per its 2026-03-30 architecture decisions. Picked over mem0 because a VC firm genuinely needs relationship history + temporal edges.

## When to pick this over `mem0.md`

| Pick Graphiti + Neo4j when… | Pick mem0 when… |
|------------------------------|-----------------|
| Relationships have time validity ("X worked at Y 2023-2025") | You just want "inject recent memories about this person" |
| Need typed edges (WORKS_AT ≠ KNOWS ≠ INTRODUCED_BY) | Plug-and-play is more important |
| Need relationship state machines (COLD → WARM → DORMANT → REACTIVATING) | Don't want to maintain a graph schema |
| Need Cypher query power | Don't need complex filter predicates |
| 20K+ people in scope (enterprise-scale) | <1K people, simple recall enough |

**Operational footprint is higher than mem0** — Docker + Flask + Python Graphiti + Node.js driver. Appropriate only for relationship-heavy use cases.

## Components

- **Graphiti core:** MIT, open-source Python library from the Zep team. Temporal abstractions baked in: every relationship has `start_date`, `end_date`, can be superseded not overwritten.
- **Neo4j Community Edition:** free, GPLv3, sufficient for single-agent deployments.
- **REST wrapper:** thin Flask app exposing Graphiti over HTTP (port 8100).
- **Node driver:** `neo4j-driver` for direct Cypher reads from OpenClaw skills.

## Install flow (from the reference snapshot's `deploy-knowledge-graph.md`)

```bash
# 1. Docker + Neo4j
docker run -d \
  --name edge-neo4j \
  -p 7474:7474 -p 7687:7687 \
  -v edge-neo4j-data:/data \
  -e NEO4J_AUTH=neo4j/<password> \
  neo4j:5-community

# 2. Python venv + Graphiti
python3 -m venv ~/.openclaw/graphiti-env
source ~/.openclaw/graphiti-env/bin/activate
pip install graphiti-core flask

# 3. REST wrapper (Flask app at :8100, launchd service)
# Full wrapper code lives in the reference snapshot's deploy-knowledge-graph.md

# 4. Node driver in workspace
cd ${WORKSPACE}/graph-tools
npm install neo4j-driver

# 5. Seed from Notion (optional — needs deploy-notion-workspace + CRM)
python seed_from_notion.py --people --companies --deduplicate
```

## Default graph schema

**Nodes:** Person, Company, Meeting, Email, Deal, ActionItem — each with properties and embedding.

**Edges (all temporal — have `start_date`, `end_date`, metadata):** `WORKS_AT`, `ATTENDED`, `KNOWS`, `INTRODUCED_BY`, `SENT_EMAIL`, `RECEIVED_EMAIL`, `COMMITTED_TO`, `PART_OF_DEAL`, `HAS_INTEREST`, `EA_FOR`.

## Relationship state machine

```
COLD → WARM → ACTIVE → DEEP → DORMANT → REACTIVATING
                ↑                           │
                └───────────────────────────┘
```

| State | How detected |
|-------|-------------|
| COLD | No prior interaction |
| WARM | 1-2 interactions in last 90 days |
| ACTIVE | 3+ interactions in last 90 days |
| DEEP | 10+ interactions total + recent activity |
| DORMANT | No interaction in 90+ days |
| REACTIVATING | New interaction after 90+ day gap |

State recomputed nightly by a scheduled job.

## Trade-offs

- ✅ Best-in-class for "relationships over time" queries
- ✅ Cypher from any skill (read path)
- ✅ Self-hosted — no SaaS dependency
- ❌ Docker + Python + Node.js = real operational footprint
- ❌ Graph schema is something you maintain
- ❌ Cypher learning curve for skill authors
- ❌ No native OpenClaw plugin — integrates via REST wrapper + Node driver

## Citations

- https://github.com/getzep/graphiti
- https://neo4j.com/download/neo4j-community-edition/
- reference architecture decisions 2026-03-30
- reference deploy-knowledge-graph.md
