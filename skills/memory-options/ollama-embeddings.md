---
name: "Ollama local embeddings"
category: "embedding-local"
repo: "https://ollama.com"
license: "MIT (Ollama runtime); per-model licenses vary"
version_last_verified: "registry snapshot 2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "auth-profiles-entry (openai-compatible endpoint)"
cost_tier: "free-local"
privacy_tier: "fully-local"
requires:
  - "Ollama installed"
  - "Sufficient RAM for model (300MB – 1.2GB)"
docs_urls:
  - "https://ollama.com/search?c=embedding"
  - "https://ollama.com/blog/embedding-models"
---

# Ollama local embeddings

Use when you want zero outbound network for embeddings, or eliminate per-call cost on a 24/7 agent. On Apple Silicon, prefer `omlx-apple-silicon.md` instead (2× faster, 50% less RAM).

## Model table (2026-04-20 registry snapshot)

Ranked by adoption:

| Model | Params | Pulls | Disk | Dims | Context | License | Tier |
|-------|--------|-------|------|------|---------|---------|------|
| [`nomic-embed-text`](https://ollama.com/library/nomic-embed-text) | 137M | 65.8M | ~300 MB | 768 | 8192 | Apache 2.0 | Workhorse — first choice for "just make it work locally" |
| [`mxbai-embed-large`](https://ollama.com/library/mxbai-embed-large) | 335M | 9.7M | ~700 MB | 1024 | 512 | Apache 2.0 | Higher quality; short context limits long docs |
| [`bge-m3`](https://ollama.com/library/bge-m3) | 567M | 3.9M | ~1.2 GB | 1024 | 8192 | MIT | Multilingual, dense+sparse+ColBERT hybrid retrieval in one model |
| `snowflake-arctic-embed` family | 22M → 335M | 3M | varies | varies | — | Apache 2.0 | Size tags for tiny (`:s`, `:xs`) through quality (`:l`) |
| [`snowflake-arctic-embed2`](https://ollama.com/library/snowflake-arctic-embed2) | ~568M | 370k | 1.2 GB | (verify) | 8K | Apache 2.0 | Multilingual, Matryoshka to 128-byte vectors |
| [`embeddinggemma`](https://ollama.com/library/embeddinggemma) | 308M | 1M | <200 MB quantized | 768 (MRL → 128) | 2048 | Gemma terms | See `embeddinggemma.md` — the legit "Gemma for embeddings" |
| `qwen3-embedding` | 0.6B / 4B / 8B | 1.8M | varies | (verify) | — | Apache 2.0 | 0.6b for CPU; 8b top-of-open-benchmark on GPU. Mem0 recommends `Qwen 600M or comparable` for hybrid search |
| `granite-embedding` | 30M / 278M | 317k | small | 384 / 768 | — | Apache 2.0 | IBM Granite — business/multilingual |
| `all-minilm` | 22M / 33M | 3M | ~50 MB | 384 | 512 | Apache 2.0 | Tiny — Raspberry Pi / edge. Quality tier: low |

### Quality tier summary

- **Top:** `qwen3-embedding:8b`, `snowflake-arctic-embed2`, `bge-m3`
- **Middle:** `mxbai-embed-large`, `nomic-embed-text`, `embeddinggemma`
- **Budget / edge:** `all-minilm`, `snowflake-arctic-embed:s`, `granite-embedding:30m`

## Install

```bash
# 1. Install Ollama if not already present
curl -fsSL https://ollama.com/install.sh | sh

# 2. Pull the embedding model
ollama pull nomic-embed-text   # or mxbai-embed-large / bge-m3 / embeddinggemma

# 3. Confirm Ollama is serving
curl -sS http://localhost:11434/api/embeddings -d '{"model":"nomic-embed-text","prompt":"test"}' | head -c 200
```

## Wire into OpenClaw

⚠️ **Schema uncertainty:** OpenClaw's `auth-profiles.json` documents Anthropic/OpenAI/Google entries. Ollama embedding provider entry shape is less clear. Attempt pattern:

```python
python3 -c "
import json, os
path = os.path.expanduser('~/.openclaw/agents/main/agent/auth-profiles.json')
with open(path) as f: auth = json.load(f)
auth['profiles']['ollama:default'] = {
    'type': 'endpoint',
    'provider': 'ollama',
    'endpoint': 'http://localhost:11434',
    'embeddingModel': 'nomic-embed-text'
}
with open(path, 'w') as f: json.dump(auth, f, indent=2)
"
openclaw memory index --force
```

If OpenClaw doesn't recognize `ollama` natively, try `openai-compatible` as the provider type.

## Dimension mismatch warning

Migrating between providers with different dimensions requires **full reindex**. Partial migration isn't possible with sqlite-vec.

## Trade-offs

- ✅ Zero outbound network
- ✅ Zero per-call cost
- ✅ Fast on Apple Silicon (use oMLX instead for 2× speedup)
- ❌ Lower quality than frontier cloud embeddings for some tasks
- ❌ Initial pull is 300MB–1.2GB per model
- ❌ RAM cost per model stays resident while Ollama is serving

## Citations

- https://ollama.com/search?c=embedding
- https://ollama.com/blog/embedding-models
- https://ollama.com/library/nomic-embed-text
- https://huggingface.co/Snowflake/snowflake-arctic-embed-l-v2.0
