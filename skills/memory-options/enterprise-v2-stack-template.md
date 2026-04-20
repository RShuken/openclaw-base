---
name: "{{AGENT_NAME}} Stack template (Apple Silicon, privacy-first, relationship-heavy)"
category: "template"
repo: "N/A (composition of other options)"
license: "N/A"
version_last_verified: "2026-04-20 (reference-snapshot deployment: 2026-03-31)"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "full stack — install order below"
cost_tier: "mostly-local"
privacy_tier: "fully-local"
requires:
  - "Apple Silicon host (M-series Mac)"
  - "16GB+ RAM"
  - "Docker"
  - "Homebrew"
docs_urls: []  # Engagement-internal docs existed but are not shipped in this public repo
---

# {{AGENT_NAME}} Stack template

Full stack deployed for {{PRINCIPAL_NAME}}'s {{AGENT_NAME}} on Mac Mini M4 Pro. Layer-by-layer, each choice has a documented reason in {{AGENT_NAME}}'s architecture decisions.

## Layer table

| # | Layer | Choice | Why | Detail |
|---|-------|--------|-----|--------|
| 1 | Local inference substrate | oMLX (not Ollama) | Apple Silicon native, 2× faster, 50% less RAM | `omlx-apple-silicon.md` |
| 2 | Background chat model | Qwen 3 8B via oMLX (reference-snapshot deployed pre-Gemma-4) — **new installs: upgrade to Gemma 4** | Free heartbeat; summarization + memory-write decisions | `gemma4-heartbeat.md` |
| 3 | Embedding provider | `mxbai-embed-large` via oMLX | nomic-embed-text rejected (install failures). VC privacy: cloud embeddings not allowed for Layer 0. | `omlx-apple-silicon.md` + `ollama-embeddings.md` for the model reference |
| 4 | Cloud embeddings | Rejected for Layer 0 | VC firm — no confidential data leaves machine. Gemini/OpenAI OAuth issues. Cloud may return for Layer 2 non-confidential work | `gemini-embeddings.md` / `openai-embeddings.md` (not used) |
| 5 | Vector storage | OpenClaw built-in sqlite-vec | Works, no reason to replace | `../deploy-memory.md` |
| 6 | Context engine | Lossless Claw with local-oMLX summary model | Compaction loss-free; summary pass $0 via local model | `lossless-claw.md` |
| 7 | Structured memory | Graphiti + Neo4j (not mem0) | Relationships have time validity; state machine COLD→WARM→ACTIVE→DEEP→DORMANT→REACTIVATING drives downstream skills | `graphiti-neo4j.md` |
| 8 | Memory tiering | Hot (<200 lines) / Warm (daily) / Cold (weekly rollups) | Operationally explicit | `community-patterns.md` |
| 9 | Heartbeat | Memory auto-save every 30 min via local Qwen 3 8B / Gemma 4 | Continuous capture; $0/fire | `community-patterns.md` + `gemma4-heartbeat.md` |
| 10 | Security | MEMORY.md never loaded in group chats / sub-agents | Baseline §K.153; enforced in AGENTS.md boot sequence | `community-patterns.md` |

## Good fit for

- Apple Silicon hosts
- Privacy-first deployments (confidential data can't leave the machine)
- Organizations with relationship-history-heavy workflows (VC firms, executive offices, sales orgs)
- 24/7 always-on agents where local-model cost of $0 matters

## Not a good fit for

- Non-Apple-Silicon hosts (oMLX is Apple-only)
- Single-user personal agents with no relationship-history needs (Neo4j overkill)
- Deployments where Docker isn't available (Graphiti needs it for Neo4j)

## Full install order

```bash
# ==========================================
# Layer 1 — oMLX substrate (see omlx-apple-silicon.md)
# ==========================================
brew tap jundot/omlx
brew install omlx

# Layer 2 — background chat model
omlx pull gemma4              # NEW installs (released 2026-04-02)
# or the reference snapshot's Qwen 3 8B:
# omlx pull qwen3:8b

# Layer 3 — embedding model
omlx pull mxbai-embed-large

# Start oMLX server (install as launchd for persistence)
omlx serve --port 8000

# ==========================================
# Layer 5 — Wire embeddings into OpenClaw (see deploy-memory.md)
# ==========================================
python3 -c "
import json, os
path = os.path.expanduser('~/.openclaw/agents/main/agent/auth-profiles.json')
with open(path) as f: auth = json.load(f)
auth['profiles']['omlx:default'] = {
    'type': 'endpoint',
    'provider': 'openai-compatible',
    'endpoint': 'http://localhost:8000/v1',
    'embeddingModel': 'mxbai-embed-large'
}
with open(path, 'w') as f: json.dump(auth, f, indent=2)
"
openclaw memory index --force

# ==========================================
# Layer 6 — Lossless Claw (see lossless-claw.md)
# ==========================================
# Verify command path first:
openclaw --help | grep -iE 'plugin|extension'
# Then (adjust based on what's available):
openclaw plugins install @martian-engineering/lossless-claw
# Configure contextEngine slot:
python3 -c "
import json, os
p = os.path.expanduser('~/.openclaw/openclaw.json')
with open(p) as f: cfg = json.load(f)
cfg.setdefault('plugins', {}).setdefault('slots', {})['contextEngine'] = 'lossless-claw'
with open(p, 'w') as f: json.dump(cfg, f, indent=2)
"
openclaw config validate

# ==========================================
# Layer 7 — Graphiti + Neo4j (see graphiti-neo4j.md)
# ==========================================
# Neo4j via Docker
docker run -d \
  --name edge-neo4j \
  -p 7474:7474 -p 7687:7687 \
  -v edge-neo4j-data:/data \
  -e NEO4J_AUTH=neo4j/<password> \
  neo4j:5-community

# Graphiti Python
python3 -m venv ~/.openclaw/graphiti-env
source ~/.openclaw/graphiti-env/bin/activate
pip install graphiti-core flask
# Deploy Flask REST wrapper as launchd service — see the reference snapshot's deploy-knowledge-graph.md

# Node.js driver in workspace
cd ${WORKSPACE}/graph-tools
npm install neo4j-driver

# ==========================================
# Layer 8 — Memory tiering (see community-patterns.md)
# ==========================================
mkdir -p ${WORKSPACE}/memory/archive
# Keep MEMORY.md <200 lines; daily notes in memory/YYYY-MM-DD.md;
# weekly rollup script archives older entries

# ==========================================
# Layer 9 — Heartbeat memory auto-save (see community-patterns.md)
# ==========================================
openclaw cron add --name heartbeat-auto-save \
  --cron "*/30 * * * *" \
  --env HEARTBEAT_TRIAGE_MODEL=omlx:gemma4 \
  --command "${WORKSPACE}/scripts/heartbeat.sh"
# heartbeat.sh calls local model via omlx endpoint;
# extracts salient facts; appends to memory/YYYY-MM-DD.md;
# writes entities to Graphiti if mentioned

# ==========================================
# Verification
# ==========================================
openclaw memory status --json | python3 -m json.tool
openclaw config get plugins.slots.contextEngine      # → "lossless-claw"
curl -sS http://localhost:7474                       # Neo4j browser up
curl -sS http://localhost:8100/health                 # Graphiti REST up
curl -sS http://localhost:8000/v1/models             # oMLX serving
openclaw cron list | grep heartbeat
```

## Citations

- reference architecture decisions 2026-03-30
- reference install checklist
- reference deploy-knowledge-graph.md
