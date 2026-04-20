---
name: "OpenAI cloud embeddings"
category: "embedding-cloud"
repo: "https://platform.openai.com/docs/guides/embeddings"
license: "OpenAI API terms"
version_last_verified: "text-embedding-3-small, text-embedding-3-large"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "auth-profiles-entry (provider=openai)"
cost_tier: "per-token-cloud"
privacy_tier: "cloud"
requires:
  - "OpenAI API key"
docs_urls:
  - "https://platform.openai.com/docs/guides/embeddings"
  - "https://developers.openai.com/api/docs/models/text-embedding-3-small"
  - "https://developers.openai.com/api/docs/models/text-embedding-3-large"
---

# OpenAI cloud embeddings

Two stable models as of 2026-04-20. No `text-embedding-4` shipped yet.

## `text-embedding-3-small` — cheapest reliable cloud

- **Dimensions:** 1536 default, Matryoshka-truncatable via `dimensions` param
- **Pricing:** **$0.02 per 1M tokens** (standard) / **$0.01 per 1M** (batch)
- **Use when:** cost-sensitive but cloud is OK. Mem0's default for a reason. Best $/quality in the cloud tier.

## `text-embedding-3-large` — higher quality, still cheap

- **Dimensions:** 3072 default, Matryoshka-truncatable
- **Pricing:** **$0.13 per 1M tokens** (standard) / **$0.065 per 1M** (batch)
- **Use when:** want top-tier retrieval quality without the preview-status risk of Gemini Embedding 2

## When to pick OpenAI

- Want stable, production-proven embeddings at known pricing
- Already have an OpenAI key from agent routing
- Want Matryoshka flexibility to trim vector size after indexing

## When to pick something else

- Need multimodal (text+image+video+audio) → `gemini-embeddings.md` (v2 preview)
- Privacy / air-gap → local options
- Want the cheapest possible and privacy matters → local
- `text-embedding-3-large` quality isn't quite enough → Gemini Embedding 2 preview

## Wire into OpenClaw

Add `openai:default` profile to `auth-profiles.json` with the key. Configure the embedding subsystem to prefer `openai`.

## Citations

- https://platform.openai.com/docs/guides/embeddings
- https://developers.openai.com/api/docs/models/text-embedding-3-small
- https://developers.openai.com/api/docs/models/text-embedding-3-large
- https://openai.com/index/new-embedding-models-and-api-updates/
