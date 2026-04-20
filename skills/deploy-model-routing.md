# Deploy Model Routing — Opinionated Recommendations

## Compatibility

- **OpenClaw Version**: 2026.4.15+
- **Not a catalog of every model.** openclaw.ai/docs/providers lists the full universe of providers; this skill intentionally does NOT mirror that. If you want model IDs and specs, read vendor docs (links at the bottom).
- **What this skill IS:** an opinionated use-case → model-tier routing guide that skills reference via the env-var convention in `_authoring/_deploy-common.md` "Per-skill env-var overrides."
- **Research basis:** all sources verified 2026-04-20. Full research notes at `audit/_model-routing-research-2026-04-20.md` (~2,800 words, 110+ citations).

## Purpose

When a skill makes an LLM call, it declares the CALL TYPE (heartbeat, synthesis, classification, coding, etc.) — not a specific model. The operator then sets an env var per skill to route that call type to a specific model. This skill is where the operator looks to pick which model.

Maintained as community-backed opinions, not a data mirror. When a new model ships that changes the recommendation for a tier, we update one row of one table here.

## Cost model / auth strategy (the first decision, before picking a tier)

**This shapes every tier recommendation below.** Before picking a model, pick how you're paying for it.

### OpenAI — subscription OAuth is the cost-efficient path

- **ChatGPT Plus / Pro / Codex subscription + OAuth into OpenClaw** → flat-rate access to Codex and GPT-5.x models. For any 24/7 agent or heavy user, subscription is dramatically cheaper than per-token API.
- OpenClaw's `openai-codex` provider supports OAuth login via `openclaw models auth login --provider openai-codex --method oauth`.
- Subscription includes generous usage caps on Codex 5.4, 5.3-codex, etc. For most operators the subscription does not throttle before end-of-month.
- **This is why Flyn's default stack uses `openai-codex/gpt-5.4` as primary** — pair with OAuth and 24/7 agent cost drops to the flat subscription price ($20-200/month) instead of pay-per-token.
- Per-token API still exists (use it for burst workloads beyond subscription caps), but the default should be OAuth.

### Anthropic — API is expensive; no equivalent subscription path

- Claude.ai subscription is for the web chat UI, **NOT API access**. No subscribe-your-way-to-unlimited like OpenAI/Codex.
- Pay-per-token API: Claude Opus 4.7 is $5 input / $25 output per 1M. For a 24/7 agent reading 100K tokens a day, that's $2.50/day ≈ $75/month minimum. Real agents blow past this fast.
- **Practical consequence:** Opus 4.7 / 4.6 are effectively price-gated for continuous agent use. Fine for one-off deep reasoning via `deploy-advisory-council.md` synthesis calls; not realistic as the primary model if you're always-on.
- For routine work, either route around Opus (use Sonnet 4.6 at $3/$15) or use an OpenAI subscription for the continuous-use load.

### Chinese frontier — dramatically cheaper than Anthropic

**Biggest shift since 2025:** Kimi K2.5, GLM-5, MiniMax M2.7, and Qwen 3.5 now score inside the top cluster on Artificial Analysis Intelligence Index, at **30-60× cheaper** than Claude Opus 4.7. MiniMax self-claims 50× cheaper input / 60× cheaper output than Opus with matching SWE-Pro. Community verdict: these are real models, not toys.

If you're considering Anthropic API for cost reasons and budget matters, the Chinese frontier is the first place to look. Accept: data travels through non-US providers (relevant if privacy is a requirement).

### Other cloud providers — per-token, usually cheaper than Anthropic

- **Google Gemini 3.1 Pro**: $2/$12 per 1M — "frontier for the price of near-frontier" (community consensus per [buildfastwithai.com](https://www.buildfastwithai.com/blogs/best-ai-models-april-2026)).
- **DeepSeek V3.2**: $0.259/$0.42 per 1M. Bulk synthesis default.
- **DeepSeek V4**: $0.30/$0.50 per 1M (released March 2026).
- **OpenRouter with `:floor`**: cheapest available model meeting your quality bar.

### Local — $0 per call

- Running oMLX / Ollama on your hardware → free after electricity. Use for heartbeat, classification, long-running background.
- Quality ceiling: local frontier in April 2026 (GLM-4.7-Flash at 59.2% SWE-bench, Qwen 3.5 235B at 991K context, Qwen3-Coder-Next r/LocalLLaMA consensus for coding agents) is genuinely usable — not a toy tier anymore.
- Cost ceiling: you pay for the hardware once.

### Decision tree for cost

```
Will your agent run continuously (24/7) or heavily (>100K tokens/day)?
├── Yes
│   ├── Need Anthropic-specific quality? → budget for per-token + accept it
│   ├── OpenAI-tier is fine? → ChatGPT/Codex subscription + OAuth (STRONGLY recommended default)
│   ├── Privacy OK with non-US providers? → Chinese frontier (Kimi / GLM / MiniMax / Qwen) at 30-60× cheaper
│   └── Want $0? → Local (oMLX / Ollama) for background; cloud frontier only for user-chat (hybrid)
└── No (occasional / burst)
    ├── Per-token API from any provider is fine
    ├── Route via OpenRouter :floor for cheapest-per-call
    └── OAuth subscription may not pay back — run the math
```

## Call types

Every LLM call in the openclaw-base skill library falls into one of these. Each has its own tier recommendation below.

| Call type | What it is | Example skills |
|-----------|-----------|----------------|
| **User-facing chat** | Operator reading the model's output directly | `deploy-identity.md` every main-session turn |
| **Synthesis** | Combine N inputs into one artifact the operator reads | `deploy-daily-briefing.md`, `deploy-advisory-council.md` synthesis, `deploy-earnings.md` |
| **Classification** | Binary / multi-class decisions at volume (is-urgent? is-spam?) | `deploy-urgent-email.md`, `deploy-security-safety.md` scan |
| **Heartbeat / background** | Scheduled low-stakes work | `deploy-health-monitoring.md`, heartbeat memory auto-save |
| **Coding** | Agents writing or modifying code | (future) dev-team delegation, autonomous coding |
| **Deep reasoning** | Hard multi-step problems worth paying for | Advisory council personas on hard cases |
| **Long-context** | Whole-codebase reads, huge docs (500K+ tokens in one shot) | Research skills, one-shot audit passes |
| **Embeddings** | Covered separately — see `memory-options/` |
| **Image / video gen** | Not text-LLM routing — see `deploy-image-gen.md` / `deploy-video-gen.md` |

## Tier recommendations

### Frontier tier (user-facing chat, synthesis, deep reasoning)

When quality matters and the human is reading the output.

| Model | Price (I/O per 1M) | Context | Why pick this |
|-------|---------------------|---------|----------------|
| **Anthropic Claude Opus 4.7** | $5 / $25 | 1M | Writing + reasoning king. 87.6% SWE-bench Verified, 64.3% SWE-bench Pro, 47% blind writing prefs vs GPT-5.4's 29%. Best when output quality is the deciding factor. **Price-gated for continuous use.** |
| **OpenAI GPT-5.4** (preferred via Codex OAuth subscription) | $2.50 / $15 API, OR flat-rate subscription | 1M+ | Strongest generalist. Codex variant leads raw coding benchmarks by single digits. **Subscription OAuth makes this the cost-efficient frontier default for 24/7 agents.** |
| **Google Gemini 3.1 Pro** | $2 / $12 | 1M–2M | "Frontier for near-frontier price." Dominates multimodal (78.2% Video-MME vs 71.4% next). |
| **xAI Grok 4** | varies | — | Intelligence Index 73 (ahead of o3 at 70, Gemini 2.5 Pro at 70, Opus 4 at 64). Leads AA Coding + Math indexes. Grok 4 Fast uses 40% fewer thinking tokens at comparable quality. |
| **GLM-5 (Reasoning)** — **open weights** | varies by host | 200K | First Chinese open-weights model realistically competing in frontier chat. AA Intelligence Index 51 (tied w/ MiniMax M2.7). 77.8% SWE-Bench Verified. Self-host or via OpenRouter. |

**Default recommendation:** OpenAI GPT-5.4 via Codex OAuth subscription. Opus only when specific output-quality tasks justify per-token cost.

### Near-frontier tier (synthesis when cost matters)

Good enough for real synthesis, cheap enough to run on every meeting / briefing / digest.

| Model | Price (I/O per 1M) | Context | Why pick this |
|-------|---------------------|---------|----------------|
| **Claude Sonnet 4.6** | $3 / $15 | 1M | Community default for "not Opus but still Claude." Best when Anthropic-style output matters. |
| **GPT-5.4-mini** | cheaper than 5.4 | — | Workhorse middle tier. Good quality/cost for synthesis batches. |
| **Gemini 2.5 Flash** | low | — | "Latency-first" flagship middle tier. Strong for tool-calling workloads. |
| **Kimi K2.5 (Reasoning)** | very cheap | 256K | 1T total / 32B active MoE. Ties Claude Opus 4.6 on AIME math (95.63%). Leads Humanity's Last Exam, BrowseComp. **Best-in-class for agentic tool-chains** — stable across 200-300 sequential tool calls. |
| **MiniMax M2.7** | $0.30 / $0.x | ~256K | 230B total / 10B active MoE. 56.22% SWE-Pro. 100 tok/s. MiniMax-claimed 50× cheaper input / 60× cheaper output than Opus, matching on SWE-Pro. Real model, not a toy. |
| **GLM-5 (non-reasoning) / GLM-4.7** | low | 200K / 128K output | 744B MoE / 40B active (GLM-5). Huge with cost-sensitive shops on Zhipu direct or OpenRouter. |
| **DeepSeek V3.2** | $0.259 / $0.42 | — | General-purpose speed tier. Top-cluster per Latent.Space. Bulk synthesis default. |
| **DeepSeek V4** | $0.30 / $0.50 | — | Released March 2026. 1T total / ~37B active. SWE-bench Verified 81% (vs V3's 69%). **Cheap frontier-grade default.** |
| **Qwen 3.5 (235B)** | low | 991K | Latent.Space's "most broadly recommended family right now across use cases." Context champion of the near-frontier. |

**Default recommendation:** Claude Sonnet 4.6 if you're Anthropic-invested; **DeepSeek V4** or **Qwen 3.5** if you're cost-optimizing; **Kimi K2.5** if your skill makes long tool-chains.

### Local tier (heartbeat, classification, background)

Models people actually run on laptops / Mac Minis to keep background cost per fire at $0. Pair with [`memory-options/omlx-apple-silicon.md`](./memory-options/omlx-apple-silicon.md) or [`memory-options/ollama-embeddings.md`](./memory-options/ollama-embeddings.md) for the runtime.

| Model | Params | Context | Why pick this |
|-------|--------|---------|----------------|
| **Gemma 4** | 5B / 8B / 4B-active MoE / 31B dense | 128K–256K | Google's local flagship. Multimodal input, native tool-use, agentic workflows, coding, NVIDIA RTX first-class. The Flyn default. |
| **Qwen 3.5 (8B / 14B / 32B)** | 8-32B | 991K | The most-recommended family, period. **Qwen 3.5 8B** is a common reference-stack pick — sweet spot for "good enough heartbeat on a Mac Mini." |
| **GLM-4.7-Flash** | 30B total / 3B active MoE | 128K | "Strongest model in the 30B class" per Zhipu. SWE-Bench Verified **59.2%** — a local model doing 59% on SWE-bench is notable. Community framing: "the local GPT-4o-mini." |
| **Mistral Small 4** | ~22B active, Apache 2.0 | 256K | Unifies Magistral (reasoning) + Pixtral (multimodal) + Devstral (agentic coding) into one. Per Mistral, matches or beats GPT-OSS 120B on several benchmarks. Great general-purpose local. |
| **Ministral 3** | 3B / 8B / 14B, each in Base/Instruct/Reasoning | — | Single-GPU deployable. Right pick when Gemma/Qwen are too heavy (laptops, edge). |
| **Llama 4 Scout** | — | **10M** | When you need "read a whole codebase" locally. Extreme context. |
| **DeepSeek R1 Distill 32B** | 32B | — | Same size as Qwen 32B but reasoning-tuned (o1-style). For local models that need to "think" before answering. |

**Default recommendation:** Qwen 3.5 8B or Gemma 4 for general heartbeat. GLM-4.7-Flash if you want the strongest local coding-capable model in 30B class.

### Coding tier (agents writing code)

| Model | Price / Availability | Why pick this |
|-------|----------------------|----------------|
| **GPT-5.4 Codex** | API + Codex subscription | Raw benchmark leader. Worth it when quality differentiates. 8-12× the price of open-weights alternatives. |
| **Claude Opus 4.7** | $5 / $25 | Leads SWE-bench Pro (64.3%) and SWE-bench Verified (87.6%). The "customer-facing codegen" quality bar. |
| **Claude Sonnet 4.6** | $3 / $15 | Claude Code community default. The "running coding agent" pick. |
| **Qwen3-Coder-Next** | open-weights, cheap cloud | **r/LocalLLaMA consensus for local coding agents.** 80B total / 3B active MoE, 256K context. Trained for long-horizon reasoning, complex tool use, recovery from execution failures. Integrates with Claude Code, Qwen Code, Kilo, Trae, Cline. |
| **Qwen3-Coder Plus** (cloud) | low | ~72% SWE-bench, 90% HumanEval+, 68% LiveCodeBench. Within 3-5 points of Sonnet/GPT-5.4 at a fraction of cost. |
| **Qwen3-Coder Flash** | **$0.10 / $0.40** | Cheapest dedicated coding model with non-toy quality. |
| **DeepSeek Coder V3 / V4** | very cheap | Top-3 for Python and C++. V4's 81% SWE-Bench Verified inherits to coding. |
| **GLM-5 (coding-tuned)** | low | SWE-Bench Verified 77.8%, Terminal-Bench 2.0 56.2%. Strong for agents that shell around a lot. |
| **MiniMax M2.5 / M2.7** | very cheap | SWE-bench Verified 80.2% / ~78%. Agent-shaped training — 80% of MiniMax's internal commits come from M2.5. |
| **Kimi K2.5 Thinking** | low | Strong on reasoning-heavy coding. |
| **Grok 4 Fast** | varies | Pre-trained on programming-rich corpus, post-trained on real pull requests. Leads AA Coding Index. |

**Default recommendation:** **Qwen3-Coder-Next** for autonomous local coding agents (community consensus). **Qwen3-Coder Flash** for cheap cloud-backed coding. **Claude Sonnet 4.6** or **GPT-5.4 Codex** when quality tops cost.

### Long-context tier (whole-codebase reads, huge docs)

| Model | Context | Note |
|-------|---------|------|
| **Llama 4 Scout** | **10M** | Practical edge for entire-codebase analysis in one pass. Local. |
| **Qwen 3.5** | **991K** | 5× Claude Opus. Open weights. |
| **Gemini 3.1 Pro / 2.5 Pro** | 1M–2M | Frontier quality throughout the whole context window. |
| **Claude Opus 4.7 / Sonnet 4.6** | 1M | Standard pricing, no tier jump. |
| **Kimi K2.5** | 256K | Less extreme than above but paired with top-tier reasoning. |
| **Qwen3-Coder-Next** | 256K | Specifically shaped for coding scaffolds. |
| **Mistral Small 4** | 256K | Local-deployable. |

**Default recommendation:** Gemini 3.1 Pro for cloud, Llama 4 Scout for local-when-you-need-10M-context.

## OpenRouter routing modes

OpenRouter is the switchboard most mixed-provider OpenClaw installs route through. Mechanics verified from [openrouter.ai/docs/guides/routing](https://openrouter.ai/docs/guides/routing/provider-selection):

| Mode | What it does | When to use |
|------|--------------|-------------|
| **Default** (no suffix) | Price-based load balancing. Prioritizes stable providers, weights by inverse square of price. Quality-preserving. | Most calls |
| **`:nitro` suffix** | Sort by throughput (tokens/sec). Disables load balancing. Example: `anthropic/claude-sonnet-4.6:nitro` | Latency-critical calls |
| **`:floor` suffix** | Sort by price, cheapest first. Example: `deepseek/deepseek-v3.2:floor` | Classification at scale, background work |
| **`openrouter/auto`** | NotDiamond-powered router. Analyzes prompt, picks from curated pool (Sonnet/Opus 4.5, GPT-5.1, Gemini 3.1 Pro, DeepSeek 3.2, rotating top performers). Returns chosen model in metadata. | Variable-quality workloads |
| **`openrouter/free`** | Randomly selects from free models, filtering by required features (vision, tool use, structured output) | Zero-cost experimentation |
| **`provider.sort = "latency"`** (API only, no shortcut) | Lowest time-to-first-token | Real-time latency requirements |
| **`partition: "none"`** | Removes per-model grouping across fallbacks. Pair with performance thresholds (e.g. ≥50 tok/sec p90). | Finding cheapest model-provider meeting a speed floor |
| **Auto Exacto** (March 2026) | Auto-reorders providers for tool-calling requests. Documented to reduce tool-call error rates **up to 88%**. | Tool-heavy OpenClaw workflows (= most of them) |

### OpenClaw integration specifics

- Slug format: `openrouter/<author>/<slug>` (example: `openrouter/moonshotai/kimi-k2.5`)
- Built-in provider support; set `OPENROUTER_API_KEY`, reference slug, done
- `openrouter/openrouter/auto` is explicitly confirmed to work with OpenClaw
- `:nitro` / `:floor` suffix handling in OpenClaw slugs is inferred to work via OpenRouter's standard passthrough — **verify in practice on first use**

## Community patterns (April 2026)

1. **"Cloud frontier for user chat, local for everything else"** — Opus 4.7 or GPT-5.4 for user-facing; Ollama/oMLX-served Qwen 3.5 8B or Gemma 4 for heartbeats, crons, embeddings, classifiers. Matches `feedback_openclaw_local_background_routing.md`.
2. **"OpenRouter `:floor` for classification"** — sub-penny per-classification at scale. Pair with quality guard: cheap model first pass, low-confidence escalates to mid-tier.
3. **"Qwen3-Coder-Next for autonomous code agents"** — r/LocalLLaMA consensus (per Latent.Space preview) for local coding agents that read files, apply patches, interpret test logs.
4. **"Multi-model cascade"** — Gemma 4 E4B (free, on-device) for simple queries → Mistral Small 4 (low cost) for complex reasoning → Llama 4 Scout (10M context) for long-context retrieval.
5. **"MiMo-V2-Pro or DeepSeek V3.2 for bulk"** — common OpenRouter pattern for batch synthesis.
6. **"GLM-4.7-Flash as the local GPT-4o-mini"** — community framing. Same niche, different sovereignty.
7. **"Kimi K2.5 for tool-chains beyond 50 calls"** — K2.5's 200-300 sequential tool-call stability is its specific brag. Pick when agent needs long unbroken tool sessions.
8. **"Qwen3-Embedding-8B over OpenAI text-embedding-3"** — MTEB multilingual #1 (70.58). Free to self-host. Default embedder of the open-source stack. See [`memory-options/ollama-embeddings.md`](./memory-options/ollama-embeddings.md).

## Anti-recommendations — what NOT to use

Models / modes that look current on older blog posts but aren't the 2026 pick:

- **MiniMax abab5.5 and below** — superseded. M2.x series is current generation.
- **Kimi K2-Instruct original** — superseded by K2-Thinking (Nov 2025) then K2.5 (Jan 2026). K3 not yet released.
- **Qwen 2.5-Coder 32B** — still works, but Qwen3-Coder-Next or Qwen3-Coder Plus is 2026's answer. Many blog posts still point here; they're stale.
- **DeepSeek V3 (non-.2)** — V3.2 and V4 both exist; V3 is legacy.
- **GLM-4.6 and below** — 4.7 Flash / GLM-5 are current.
- **Llama 3.3 / 3.4** — Llama 4 Scout / Maverick is where Meta is in 2026.
- **GPT-4o / GPT-4o-mini** — superseded by GPT-5.4 family. GPT-5.4-nano at $0.20/$1.25 replaces mini for classification.
- **Claude 3.5 Sonnet / Haiku 3.5** — superseded by 4.6 / 4.5.
- **Gemini 2.5 Pro as "latest flagship"** — Gemini 3.1 Pro is the current top per April 2026 benchmarks.
- **o1 / o1-mini** — DeepSeek R1 / R2 are ~96% cheaper on output with comparable reasoning. Only pick o1 for OpenAI-ecosystem lock-in.
- **Raw `openrouter/auto` without cost guardrail** — great for variable workloads, but for high-volume single-task pipelines, pinning to `model:floor` gives predictable cost. Community warning.

## How skills use this guide

### In the skill's compat block

Instead of naming a specific model:

```markdown
**Model**: makes `classification` calls — see `deploy-model-routing.md` for recommended tiers. Override via `URGENT_EMAIL_MODEL` env var.
```

### In the skill's install / cron example

```bash
openclaw cron add --name urgent-email \
  --cron "*/5 * * * *" \
  --env URGENT_EMAIL_MODEL=openrouter/deepseek/deepseek-v3.2:floor \
  --command "${WORKSPACE}/scripts/urgent-email-scan.sh"
```

The env var value comes from THIS guide's tier tables.

### In the skill's body

Use the bash pattern from [`_authoring/_deploy-common.md`](./_authoring/_deploy-common.md):

```bash
MODEL_FLAG=""
if [ -n "${URGENT_EMAIL_MODEL:-}" ]; then
  MODEL_FLAG="--model ${URGENT_EMAIL_MODEL}"
fi
openclaw agent --agent main $MODEL_FLAG -m "<the prompt>"
```

If the env var is unset, defers to the agent default.

## Vendor docs — authoritative sources for current model IDs + pricing

Always prefer these for current specs:

- **Anthropic**: [platform.claude.com/docs/about-claude/pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- **OpenAI**: [platform.openai.com/docs/models](https://platform.openai.com/docs/models) + [developers.openai.com/codex/models](https://developers.openai.com/codex/models)
- **Google Gemini**: [ai.google.dev/gemini-api/docs/models](https://ai.google.dev/gemini-api/docs/models) + [ai.google.dev/gemini-api/docs/pricing](https://ai.google.dev/gemini-api/docs/pricing)
- **xAI Grok**: [x.ai/news](https://x.ai/news) (model release notes)
- **Moonshot (Kimi)**: [platform.moonshot.ai](https://platform.moonshot.ai/)
- **Zhipu (GLM)**: [docs.z.ai/guides/llm/glm-4.7](https://docs.z.ai/guides/llm/glm-4.7) + [huggingface.co/zai-org](https://huggingface.co/zai-org)
- **Qwen**: [qwen.ai/blog](https://qwen.ai/blog) + [huggingface.co/Qwen](https://huggingface.co/Qwen)
- **DeepSeek**: [deepseek.ai/blog](https://deepseek.ai/blog)
- **MiniMax**: [minimax.io/news](https://www.minimax.io/news)
- **Mistral**: [mistral.ai/news](https://mistral.ai/news)
- **Meta Llama**: [ai.meta.com/llama](https://ai.meta.com/llama)
- **Gemma**: [ai.google.dev/gemma/docs](https://ai.google.dev/gemma/docs) + [ollama.com/library/gemma4](https://ollama.com/library/gemma4)
- **OpenRouter**: [openrouter.ai/docs/guides/routing](https://openrouter.ai/docs/guides/routing/provider-selection)
- **Benchmark reference**: [artificialanalysis.ai](https://artificialanalysis.ai) (cross-provider) + [lmcouncil.ai/benchmarks](https://lmcouncil.ai/benchmarks)

## Platform caveats — OpenClaw runtime model resolution

A model ID being valid at the vendor is not the same as OpenClaw's runtime resolving it to the expected context window or feature set. Before trusting a default, run `openclaw models list --all` on the target machine and confirm the provider/model pair actually resolves.

Tracked OpenClaw issues (re-check status before treating as current):
- [openclaw/openclaw#37623](https://github.com/openclaw/openclaw/issues/37623) — `openai-codex/gpt-5.4` was configurable but resolved as "Unknown model" in older releases.
- [openclaw/openclaw#55461](https://github.com/openclaw/openclaw/issues/55461) — OpenClaw 2026.3.24 resolved `openai-codex/gpt-5.4` to ~266k context unless a `models.providers` override set the correct `contextTokens`.
- [openclaw/openclaw#36817](https://github.com/openclaw/openclaw/issues/36817) — tracking issue for gpt-5.4 availability / support.

Per-deployment override pattern when `openclaw models list` shows `Unknown model` or a truncated context window:

```json
{
  "models": {
    "providers": {
      "openai-codex": {
        "models": [
          { "id": "gpt-5.4", "contextTokens": 1050000 }
        ]
      }
    }
  }
}
```

### Codex-family migration note

Per OpenAI's GPT-5.4 launch: `gpt-5.4` replaces `gpt-5.2` in the API and `gpt-5.3-codex` in Codex. The Codex-specific IDs still exist and still work, but `gpt-5.4` is the path forward for new Codex routing — and the flat-rate OAuth subscription (§ "Cost model") covers it.

## Caveats on the research behind this skill

From `audit/_model-routing-research-2026-04-20.md`:

- **Reddit primary threads not directly fetched** — community-sentiment signal is second-hand via Latent.Space, BenchLM, and aggregator blogs. Direction is reliable; exact percentages are soft.
- **docs.openclaw.ai returned ECONNREFUSED** during research — OpenClaw-side integration details came from OpenRouter's `openrouter.ai/docs/guides/coding-agents/openclaw-integration`, not OpenClaw's own docs. Verify against live `openclaw models list` on your target.
- **Kimi K3 was not found** — K2.5 + K2-Thinking is current state as of 2026-04-20.
- **Grok 4.20 / 4.3** appear in some sources; treat Grok 4 as the safe current reference.
- **Vendor benchmark claims are vendor-reported** unless cross-confirmed by Artificial Analysis or LMCouncil.

## Dependencies

- **Referenced by:** every skill in the library that makes LLM calls (see `_authoring/_deploy-common.md` Per-skill env-var overrides)
- **Complements:** `memory-options/` (covers embedding model routing separately)
- **References:** vendor docs for specs, Artificial Analysis / LMCouncil for benchmarks, Reddit / Latent.Space for community consensus
- **Research provenance:** `audit/_model-routing-research-2026-04-20.md` (full citations)
