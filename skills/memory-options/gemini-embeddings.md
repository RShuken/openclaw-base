---
name: "Gemini cloud embeddings"
category: "embedding-cloud"
repo: "https://ai.google.dev/gemini-api/docs/embeddings"
license: "Google API terms"
version_last_verified: "gemini-embedding-2-preview, gemini-embedding-001"
last_checked: "2026-04-20"
maintenance_status: "active (v1 stable; v2 public preview since 2026-03-10)"
openclaw_integration: "auth-profiles-entry (provider=google)"
cost_tier: "per-token-cloud"
privacy_tier: "cloud"
requires:
  - "Gemini API key"
docs_urls:
  - "https://ai.google.dev/gemini-api/docs/embeddings"
  - "https://ai.google.dev/gemini-api/docs/models/gemini-embedding-2-preview"
  - "https://ai.google.dev/gemini-api/docs/pricing"
---

# Gemini cloud embeddings

Two shipped models as of 2026-04-20 — **verified via direct fetch on [ai.google.dev/gemini-api/docs/embeddings](https://ai.google.dev/gemini-api/docs/embeddings)**.

## `gemini-embedding-2-preview` — current flagship, #1 MTEB

- **Status:** Public preview since 2026-03-10. Production-integration allowed but subject to iterative refinement before GA
- **Modalities:** **text, images, video, audio, PDFs** — single unified embedding space. First natively multimodal embedding model ([Google blog](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/))
- **Architecture:** built on the Gemini foundation model itself (not a separate encoder with pooling)
- **Dimensions:** 3072 default. MRL-flexible 128-3072; recommended sizes: **768, 1536, 3072**
- **Context:** 8,192 tokens per input (4× larger than v1)
- **Languages:** 100+
- **Benchmark:** #1 MTEB ranking as of release ([VentureBeat](https://venturebeat.com/data/googles-gemini-embedding-2-arrives-with-native-multimodal-support-to-cut/))
- **Access:** Gemini API (model ID: `gemini-embedding-2-preview`) + Vertex AI
- **Pricing:** preview — verify current pricing on the docs page, may change before GA

**Why this matters for memory:** multimodal means you can embed images, meeting videos, audio transcripts, and PDFs into the SAME vector space as text notes. Retrieval can find "that slide from the Sequoia deck" with a text query. Huge upgrade over v1 for media-rich workspaces.

**Caveat:** preview status. Don't build critical production on an API that might change before GA. For personal use: fine. For a paying client deployment: stick with v1 until GA announced.

## `gemini-embedding-001` — stable, production

- **Status:** Stable since June 2025. Production-safe choice when preview isn't acceptable.
- **Default in `deploy-memory.md`** — the baseline openclaw-base ships with this
- **Modalities:** text only
- **Dimensions:** 3072 default. MRL-flexible 128-3072; recommended: **768, 1536, 3072**
- **Context:** 2,048 tokens per input — hard truncation for longer files
- **Per-request limits:** 250 input texts max · 20,000 tokens total
- **Pricing:** $0.15 per 1M input tokens (standard); $0.075 per 1M (batch)
- **Rate limits:** tokens-per-minute per project. Free tier exists but is the first thing to exhaust on a 24/7 agent

## When to pick Gemini

- Want best retrieval quality and willing to pay for it
- Need multimodal (images/video/audio/PDFs in same space) → Gemini Embedding 2 preview is the only mainstream option
- Already on the Google ecosystem

## When to pick something else

- Privacy / air-gap hard requirement → local (`embeddinggemma.md`, `ollama-embeddings.md`, `omlx-apple-silicon.md`)
- 24/7 agent hitting free quota → OpenAI `text-embedding-3-small` is cheaper at $0.02/M
- Production-critical and preview feels risky → use `gemini-embedding-001` or `openai-embeddings.md`

## Wire into OpenClaw

Add `google:default` profile to `auth-profiles.json` with the Gemini API key. The memory subsystem auto-detects Google provider from the profile — see `deploy-memory.md` for the exact Python script.

## Citations

- https://ai.google.dev/gemini-api/docs/embeddings
- https://ai.google.dev/gemini-api/docs/models/gemini-embedding-2-preview
- https://ai.google.dev/gemini-api/docs/pricing
- https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/
