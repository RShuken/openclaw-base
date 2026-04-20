---
name: "EmbeddingGemma"
category: "embedding-local"
repo: "https://huggingface.co/google/embeddinggemma-300m"
license: "Gemma license terms (NOT Apache/MIT)"
version_last_verified: "google/embeddinggemma-300m"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "auth-profiles-entry via Ollama or oMLX endpoint"
cost_tier: "free-local"
privacy_tier: "fully-local"
requires:
  - "Ollama or oMLX runtime"
docs_urls:
  - "https://ai.google.dev/gemma/docs/embeddinggemma"
  - "https://huggingface.co/google/embeddinggemma-300m"
  - "https://developers.googleblog.com/en/introducing-embeddinggemma/"
  - "https://arxiv.org/abs/2509.20354"
---

# EmbeddingGemma

## Disambiguation

Short answer: **use `EmbeddingGemma`, not raw Gemma 2 / Gemma 3 / Gemma 4 chat weights** for embeddings. Chat-family Gemma models can be mean-pooled for embeddings but quality is well below the purpose-built embedding model + you'd be stacking license risk.

`EmbeddingGemma` is **NOT the same as `gemini-embedding-001`** — see `gemini-embeddings.md` for the cloud sibling.

## Specs

- **Parameters:** 308M
- **Architecture:** built on Gemma 3 transformer backbone, **converted to bi-directional attention** (encoder, not decoder). Legit embedding model, not a chat model with pooling.
- **Dimensions:** 768 default; Matryoshka-reducible to 512 / 256 / 128
- **Context:** 2K tokens per input
- **Footprint:** <200 MB RAM quantized, <22 ms on EdgeTPU. Designed for on-device.
- **MTEB:** highest-ranking open multilingual embedding model under 500M params per Google's claim (verify independently)
- **License:** Gemma license terms — permissive for most use, but **not Apache / not MIT**. Read it before commercial deployment.

## Install

```bash
ollama pull embeddinggemma
# or on Apple Silicon
omlx pull embeddinggemma
```

## Wire into OpenClaw

Same pattern as `ollama-embeddings.md` / `omlx-apple-silicon.md` — just substitute the model name.

## EmbeddingGemma vs `gemini-embedding-001` at a glance

| | EmbeddingGemma | `gemini-embedding-001` |
|---|---|---|
| Where it runs | Local (Ollama/oMLX) | Cloud (Google API) |
| Dimensions | 768 (MRL to 128) | 3072 (MRL to 768/1536) |
| License | Gemma terms | Google API terms |
| Cost | $0 compute | $0.15 per 1M tokens |
| Privacy | Local only | Google sees text |
| Quality tier | Middle | Top cloud tier (though Gemini Embedding 2 Preview is #1 MTEB now — see `gemini-embeddings.md`) |

## Trade-offs

- ✅ Matryoshka lets you trade quality for DB size (128-dim vectors = 6× smaller than 768-dim)
- ✅ Tiny quantized footprint — runs on laptops, mobile, edge
- ❌ Gemma license means commercial use needs a read-through
- ❌ Short 2K context — long memory files get truncated

## Citations

- https://ai.google.dev/gemma/docs/embeddinggemma
- https://developers.googleblog.com/en/introducing-embeddinggemma/
- https://huggingface.co/google/embeddinggemma-300m
- https://arxiv.org/abs/2509.20354
