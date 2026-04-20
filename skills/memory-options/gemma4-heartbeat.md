---
name: "Gemma 4 (local chat / heartbeat / tool-call model)"
category: "chat-heartbeat"
repo: "https://deepmind.google/models/gemma/gemma-4/"
license: "Gemma terms"
version_last_verified: "gemma4 (released 2026-04-02)"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "via --model flag per skill env vars (e.g. HEARTBEAT_TRIAGE_MODEL)"
cost_tier: "free-local"
privacy_tier: "fully-local"
requires:
  - "Ollama or oMLX runtime"
docs_urls:
  - "https://deepmind.google/models/gemma/gemma-4/"
  - "https://ai.google.dev/gemma/docs"
  - "https://ollama.com/library/gemma4"
---

# Gemma 4 — recommended local heartbeat model

**This is NOT an embedding model.** Gemma 4 is a chat/reasoning/tool-use model. Memory workloads need both an embedding model (for vector storage) AND a chat model (for heartbeats, entity extraction, summary generation, tool-use). This file covers the chat model — pair with `ollama-embeddings.md` or `omlx-apple-silicon.md` for the embedding side.

## Why Gemma 4 as the default heartbeat model

- Released 2026-04-02 — fresh, forward-looking
- Multimodal input (text + audio + image) — heartbeat can analyze screenshots or meeting audio
- 128K-256K context — full session history in one shot without chunking
- Native tool-use — heartbeat can call `memory_store` / Graphiti write directly instead of just emitting text
- Runs on both Ollama (all platforms) and MLX (Apple Silicon) — cross-platform for mixed fleets

## Variants

| Model tag | Size | When |
|-----------|------|------|
| `gemma4:e4b` | 4B-effective params | Tight memory (laptops, <8GB free) |
| `gemma4` (default) | Larger (verify exact on Ollama) | Standard heartbeat on 16GB+ |
| `gemma4:27b` (verify availability) | 27B | High-quality heartbeat on GPU/unified-memory |

Full size table: verify on [ollama.com/library/gemma4](https://ollama.com/library/gemma4).

## Install

```bash
# Ollama (cross-platform)
ollama pull gemma4           # default size
ollama pull gemma4:e4b       # 4B-effective for tight memory

# oMLX (Apple Silicon — 4C)
omlx pull gemma4             # MLX-converted (text-only for now)
```

## Wire into OpenClaw heartbeat

Per the skill env var convention (`_authoring/_deploy-common.md` "Per-skill env-var overrides"):

```bash
# Set in the heartbeat cron
openclaw cron add --name heartbeat --cron "0 */30 * * *" \
  --env HEARTBEAT_TRIAGE_MODEL=ollama/gemma4 \
  --command "${WORKSPACE}/scripts/heartbeat.sh"
```

Or for the advisory council persona calls:

```bash
openclaw cron add --name advisory-nightly --cron "30 4 * * *" \
  --env COUNCIL_PERSONA_MODEL=ollama/gemma4:e4b \
  --env COUNCIL_SYNTHESIS_MODEL=openai-codex/gpt-5.4 \
  --command "${WORKSPACE}/scripts/advisory-council.sh"
```

## Known caveats

- **Tool-use format quirks in reasoning mode.** Per [haimaker.ai guide](https://haimaker.ai/blog/gemma-4-ollama-openclaw-setup/), Gemma 4 in reasoning mode can emit tool calls in slightly off-spec formats. Workaround: disable reasoning mode for tool-calling heartbeats (use the non-reasoning variant) or stick with Qwen 3 8B for tool-call-heavy loops until Gemma 4 tool-format settles.
- **Gemma license ≠ Apache/MIT.** Permissive for most use but commercial deployments should read it.
- **MLX conversion may lag Ollama for new size variants** — check availability before assuming every size is on MLX.

## Alternatives

| Model | When to pick over Gemma 4 |
|-------|----------------------------|
| `qwen3:8b` | the reference snapshot's {{AGENT_NAME}} choice (deployed 2026-03-31, pre-Gemma-4). Tool-call formats more stable in reasoning mode. |
| `llama3.1:8b` / `llama3.3` | Bigger ecosystem of fine-tunes; slightly older by 2026 standards |
| `mistral-small3` | Smaller footprint when RAM is tight |
| `glm-4.7-flash` | Community-noted fast local pick (baseline §N) |

## Decision table

| Constraint | Pick |
|------------|------|
| Apple Silicon + multimodal heartbeat | `omlx:gemma4` |
| Apple Silicon + text-only | `omlx:gemma4` or `omlx:qwen3:8b` |
| Linux/Windows + multimodal | `ollama pull gemma4` |
| Linux/Windows + text-only | `ollama pull gemma4` or `ollama pull qwen3:8b` |
| <8GB RAM available | `gemma4:e4b` or `qwen3:0.6b` or `mistral-small3` |
| Need tool-calls stable in reasoning mode today | `qwen3:8b` (Gemma 4 reasoning-mode tool-call format issues as of 2026-04) |

## Citations

- https://deepmind.google/models/gemma/gemma-4/
- https://ai.google.dev/gemma/docs
- https://ollama.com/library/gemma4
- https://haimaker.ai/blog/gemma-4-ollama-openclaw-setup/
