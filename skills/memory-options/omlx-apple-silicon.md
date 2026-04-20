---
name: "oMLX (Apple Silicon local runtime)"
category: "embedding-local chat-heartbeat"
repo: "https://github.com/jundot/omlx"
license: "MIT (runtime); per-model licenses vary"
version_last_verified: "brew tap jundot/omlx (verify current version)"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "auth-profiles-entry (openai-compatible endpoint)"
cost_tier: "free-local"
privacy_tier: "fully-local"
requires:
  - "Apple Silicon (M1/M2/M3/M4/M4 Pro/Max/Ultra)"
  - "Homebrew"
docs_urls:
  - "https://github.com/jundot/omlx"
---

# oMLX — Apple Silicon alternative to Ollama

**Use this instead of Ollama on any Apple Silicon host.** Native Metal acceleration gives 2× inference speed and ~50% less RAM for the same model. This is the substrate the reference snapshot's {{AGENT_NAME}} uses on its Mac Mini M4 Pro.

## When to pick oMLX over Ollama

- Target is Apple Silicon (M-series Mac)
- Want max throughput on local models
- Want OpenAI-compatible endpoint for simple OpenClaw wiring
- Running heartbeat + embedding on the same machine and RAM is tight

## When to stick with Ollama

- Host is Linux or Windows
- Need a model variant that's only on Ollama's registry and not yet MLX-converted

## Architecture

- Metal-accelerated inference (MLX) — uses Apple's unified memory architecture natively. No CPU↔GPU copies like llama.cpp does
- OpenAI-compatible API at `http://localhost:8000/v1` — same shape as Ollama's `/v1`
- Supports MLX-converted versions of most open models: Qwen 3, Gemma 4, Llama 3.x, embedding models

## Performance (reference engagement bench)

Qwen 3 8B on M4 Pro:
- oMLX: ~80 tokens/s
- Ollama: ~40 tokens/s

Roughly 2×. Embedding throughput improvement is similar.

## Install

```bash
brew tap jundot/omlx
brew install omlx

# Pull models
omlx pull mxbai-embed-large   # embeddings
omlx pull qwen3:8b            # chat / heartbeat
omlx pull gemma4              # newer alternative (2026-04-02 release)

# Start server (or install as launchd service)
omlx serve --port 8000
```

## Verify

```bash
curl -sS http://localhost:8000/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"mxbai-embed-large","input":"test"}' | head -c 300
```

## Wire into OpenClaw

```python
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
```

oMLX speaks OpenAI API shape — use `openai-compatible` as the provider type if OpenClaw doesn't recognize `omlx` natively.

## Trade-offs vs Ollama

- ✅ 2× faster on Apple Silicon
- ✅ ~50% less RAM
- ✅ Same model catalog (MLX-converted versions on HuggingFace)
- ✅ OpenAI-compatible endpoint = easier OpenClaw integration
- ❌ Apple Silicon only
- ❌ Smaller community than Ollama (fewer blog posts, fewer Stack Overflow answers)
- ❌ MLX model conversions sometimes lag Ollama by days-to-weeks for new releases

## Why the reference engagement rejected Ollama for {{AGENT_NAME}}

Per 2026-03-30 architecture decisions:

- `nomic-embed-text` failed to install on multiple client systems (documented Ollama bug — crashes, 50/50 API failures)
- Ollama's RAM footprint on M4 Pro was 2× what oMLX used for the same models
- oMLX's Metal acceleration is a first-class requirement for 24/7 agent cost optimization

## Citations

- https://github.com/jundot/omlx
- reference architecture decisions 2026-03-30
