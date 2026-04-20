# Community Model Routing Research — 2026-04-20

## Source caveats

- **Reddit primary threads not directly fetched.** Google web search surfaced summaries of r/LocalLLaMA consensus ("overwhelming consensus" for Qwen3-Coder-Next, etc.) but I did not pull raw Reddit thread URLs — much of the community-sentiment signal is second-hand via Latent.Space, BenchLM, and aggregator blogs. Treat direction as reliable, treat exact percentages as soft.
- **docs.openclaw.ai returned ECONNREFUSED** during fetching — OpenClaw-side integration details are from OpenRouter's `openrouter.ai/docs/guides/coding-agents/openclaw-integration` page and community setup guides (skywork.ai, verdent.ai), not OpenClaw's own docs.
- **The "47 providers" figure in the prompt** did not confirm in fetched sources. OpenRouter's OpenClaw integration page only confirms OpenRouter as one of several supported providers; third-party sources reference "Arcee AI bundled plugin" and Trinity catalog entries, but do not enumerate all 47.
- **Latent.Space "Top Local Models — April 2026"** is paywalled past the preview; only the top-six list and coding consensus are publicly visible.
- **Kimi K2.5 was released January 2026** per InfoQ/Moonshot. Kimi K3 was **not found** in any searched source as of 2026-04-20 — K2.5 + K2-Thinking is the current state.
- **Grok 4.20 / 4.3** appear in sources but the exact current-as-of-April-2026 version is ambiguous; treat Grok 4 as the safe current-flagship reference.
- **MiMo-V2-Pro** and GPT-oss 20B appear in multiple aggregator sources but I did not deep-verify — noted where relevant.

---

## Frontier tier (user-facing chat, synthesis, deep reasoning)

These are what people put in the "the user is literally talking to me right now" seat.

- **Anthropic Claude Opus 4.7** — Still the writing/reasoning king. Leads SWE-bench Pro at 64.3%, SWE-bench Verified at 87.6%, and wins 47% of blind writing prefs vs GPT-5.4's 29% and Gemini 3.1's 24%. $5/$25 per M tokens, 1M context. Community's pick when output quality is the deciding factor.
- **OpenAI GPT-5.4** — The strongest generalist; sits middle on price at $2.50/$15. Codex variant (GPT-5.4 Codex) is the raw-benchmark coding leader by single-digit points but costs 8-12x more than Qwen3-Coder Plus.
- **Google Gemini 3.1 Pro** — Best cost/performance in the frontier cluster at $2/$12. Dominates multimodal (Video-MME 78.2% vs 71.4% next best). 1M-2M context. Community consensus: "frontier for the price of near-frontier."
- **xAI Grok 4** — Artificial Analysis Intelligence Index 73 (ahead of o3's 70, Gemini 2.5 Pro's 70, Opus 4's 64). Leads their Coding Index and Math Index. ARC-AGI V2 at 15.9% (nearly 2x Opus). Grok 4 Fast uses 40% fewer thinking tokens with comparable quality.
- **GLM-5 (Reasoning)** — Open-weights frontier. Artificial Analysis puts GLM-5.1 (Reasoning) at Intelligence Index 51, GLM-5 (Reasoning) at 50, tied with MiniMax M2.7. SWE-Bench Verified 77.8%. This is the first Chinese open-weights model realistically competing in the frontier chat seat.

---

## Near-frontier tier (synthesis when cost matters)

The "good enough for real synthesis, cheap enough to run on every meeting" sweet spot.

- **Claude Sonnet 4.6** — $3/$15, full 1M context at standard pricing. Community default for "not Opus but still Claude." Best when you need Anthropic-style output but can't pay Opus rates.
- **GPT-5.4-mini** — The workhorse middle. Good quality/cost for synthesis batches.
- **Gemini 2.5 Flash** — The "latency-first" flagship middle tier. Particularly strong for tool-calling workloads.
- **Kimi K2.5 (Reasoning)** — 1T total / 32B active MoE, 256K context. Ties Claude Opus 4.6 on AIME math at 95.63%. Leads on Humanity's Last Exam, BrowseComp. Best-in-class for agentic tool-chains (stable across 200-300 sequential tool calls). Strong math and corporate finance reasoning; weaker on terminal tasks.
- **MiniMax M2.7** — The cost-leader of near-frontier. 230B total / 10B active MoE. SWE-Pro 56.22%, 100 tok/s, $0.30/M input. Self-claim: 50x cheaper than Opus on input, 60x cheaper on output, matches on SWE-Pro, 3x faster. Community verdict: real model, not a toy.
- **GLM-5 (non-reasoning) / GLM-4.7** — 744B MoE / 40B active (GLM-5), 200K context, 128K output. Hugely popular with cost-sensitive shops on Zhipu direct or OpenRouter. GLM-5 released Feb 2026 with ~30% price bump over 4.7.
- **DeepSeek V3.2** — General-purpose speed tier at $0.259/$0.42 per M tokens. Still firmly "top cluster" per Latent.Space top-6 list. Goes into bulk-synthesis buckets on most cost-optimized routes.
- **DeepSeek V4** — Released early March 2026. 1T total / ~37B active, $0.30/$0.50, SWE-bench Verified 81% (vs V3's 69%). Now the default "cheap frontier-grade" pick.
- **Qwen 3.5 (235B)** — Latent.Space's "most broadly recommended family right now across use cases." Context champion at 991K tokens on Qwen 3.5.

---

## Local tier (heartbeat, classification, background)

Models people actually run on laptops and Mac Minis to keep the background-cost-per-fire at $0.

- **Gemma 4** — Google's local flagship. Sizes ~5B dense, 8B dense, 26B total / 4B active MoE, 31B dense. Positioned for reasoning, agentic workflows, coding, multimodal. NVIDIA RTX/DGX Spark first-class. Community's preferred "Google-style, local" pick.
- **Qwen 3.5 (8B / 14B / 32B)** — Broadly the most-recommended family period. The 8B is the operator's {{FAMILY_NAME}} pick and matches the community sweet-spot for "good enough heartbeat on a Mac Mini."
- **GLM-4.7-Flash** — 30B total / 3B active MoE, 128K context. Explicitly marketed as "strongest model in the 30B class" for local deployment. SWE-Bench Verified 59.2% (a local model doing 59% is notable). Apache-style weights available.
- **Mistral Small 4** — ~22B active, Apache 2.0, unifies Magistral (reasoning) + Pixtral (multimodal) + Devstral (agentic coding) capabilities into one model. 256K context. Per Mistral's own eval it matches or surpasses GPT-OSS 120B on several key benchmarks. Hybrid chat/code/agent — good general-purpose local.
- **Ministral 3 (3B / 8B / 14B)** — Each size comes in Base / Instruct / Reasoning variants. Single-GPU deployable, targets laptops and edge. The right pick when Gemma/Qwen are too heavy.
- **Llama 4 Scout** — 10M context window, ideal for "read a whole codebase locally" cases. Per aggregator sources, it's the go-to when you need huge context on-device.
- **DeepSeek R1 Distill 32B** — Same size as Qwen 32B but reasoning-tuned (o1-style). For cases where the local model needs to "think" before answering.
- **Qwen3-Embedding (0.6B / 4B / 8B)** — Flexible dimensions from 32 to 4096. Qwen3-Embedding-8B ranks #1 on MTEB multilingual (score 70.58). NV-Embed-v2 is the other top self-hosted pick. Default local embedder for OpenClaw heartbeats, KB, memory.

---

## Coding tier (for agents writing code)

- **GPT-5.4 Codex** — Raw benchmark leader. Worth the price for customer-facing codegen where quality is the differentiator. 8-12x the price of open-weights alternatives.
- **Claude Sonnet 4.6 / Opus 4.7** — Claude Code community default. Opus 4.7 leads SWE-bench Pro (64.3%) and SWE-bench Verified (87.6%). Sonnet 4.6 is the "running coding agent" default.
- **Qwen3-Coder-Next** — r/LocalLLaMA "overwhelming consensus" for local coding (per survey summaries). 80B total / 3B active MoE, 256K context, released Feb 2026. Specifically trained for "long-horizon reasoning, complex tool usage, and recovery from execution failures." Integrates with Claude Code, Qwen Code, Kilo, Trae, Cline. This is the pick when you want local autonomous coding.
- **Qwen3-Coder Plus** — Cloud version. ~72% SWE-bench, 90% HumanEval+, 68% LiveCodeBench, 65% Aider Polyglot. Within 3-5 points of Sonnet/GPT-5.4 on some benchmarks at a fraction of the cost.
- **Qwen3-Coder Flash** — $0.10/$0.40 per M tokens. Cheapest dedicated coding model with non-toy quality.
- **DeepSeek-Coder V3 / V4** — DeepSeek-Coder-V3 remains "top-3 for Python and C++" per aggregator sources. V4 coding upgrade is implicit in the 81% SWE-Bench Verified jump.
- **GLM-4.7** / **GLM-5** (coding-tuned) — GLM-5 SWE-Bench Verified 77.8%, Terminal-Bench 2.0 56.2%. Strong choice for agents that shell around a lot. GLM-4.7-Flash locally for $0-per-call agent dev.
- **MiniMax M2.5 / M2.7** — SWE-bench Verified M2.5 80.2%, M2.7 ~78%. M2.5-generated code is 80% of MiniMax's own internal commits. Agent-shaped training.
- **Kimi K2.5** — Thinking variant is strong on reasoning-heavy coding; non-reasoning K2.5 is "strong open-weight deployment option" (community verdict).
- **Grok 4 / Grok Fast** — Grok Fast was pre-trained on a "programming-rich corpus and post-trained on real pull requests." Leads AA Coding Index.

---

## Long-context tier (whole-codebase reads, huge docs)

- **Llama 4 Scout** — 10M context. Practical edge for entire-codebase analysis in one pass.
- **Qwen 3.5** — 991K context, nearly 5x Opus. Context champion of the Chinese frontier.
- **Gemini 3.1 Pro / 2.5 Pro** — 1M-2M context, frontier quality throughout.
- **Claude Opus 4.7 / Sonnet 4.6** — 1M context at standard pricing (no tier jump).
- **Kimi K2-Thinking / K2.5** — 256K native INT4 quant. Less extreme than Llama 4 Scout but paired with top-tier reasoning.
- **GLM-5** — 200K context, 128K output.
- **Qwen3-Coder-Next** — 256K context specifically shaped for coding scaffolds.
- **Mistral Small 4** — 256K, local-deployable.

---

## OpenRouter routing modes

OpenRouter is the switchboard most mixed-provider OpenClaw installs route through. Key mechanics (verified from openrouter.ai/docs/guides/routing/):

- **Default (no suffix)** — Price-based load balancing. Prioritizes providers with no outages in last 30 seconds, weights selection by inverse square of price among stable providers, uses remaining providers as fallbacks. This is *not* the same as `:floor` — it's quality-preserving load balancing.
- **`:nitro` suffix** — Sort by throughput (tokens/sec). Disables load balancing; providers tried sequentially by speed. Example: `anthropic/claude-sonnet-4.6:nitro`.
- **`:floor` suffix** — Sort by price, cheapest first. Disables load balancing. Example: `deepseek/deepseek-v3.2:floor`. Use for background classification at scale.
- **`provider.sort = "latency"`** — Third sort option (API config only, no shortcut suffix). Lowest time-to-first-token.
- **`openrouter/auto` (Auto Router)** — Powered by NotDiamond. Analyzes prompt, picks from curated pool (Claude Sonnet/Opus 4.5, GPT-5.1, Gemini 3.1 Pro, DeepSeek 3.2, and rotating top performers). Returns which model it picked in response metadata.
- **`openrouter/free`** — Router that randomly selects from free models, filtering for needed features (vision, tool calling, structured output).
- **Advanced `partition: "none"`** — Removes per-model grouping when using fallbacks, sorts endpoints globally across all models. Pair with performance thresholds (e.g. ≥50 tok/sec p90) to find cheapest model-provider combo meeting a speed floor.
- **Auto Exacto (March 2026)** — OpenRouter feature that automatically re-orders providers for tool-calling requests; documented to reduce tool-call error rates up to 88%. Relevant to any OpenClaw install because OpenClaw is tool-heavy.

**OpenClaw integration specifics:** Slug format is `openrouter/<author>/<slug>` (example: `openrouter/moonshotai/kimi-k2.5`). Built-in support; set `OPENROUTER_API_KEY` env var, reference the slug, done. OpenRouter's OpenClaw doc explicitly confirms `openrouter/openrouter/auto` works. OpenRouter's OpenClaw doc does NOT mention `:nitro` or `:floor` — treat their use in OpenClaw slugs as "should work via OpenRouter's standard suffix handling" but verify in practice.

---

## Community patterns worth citing

1. **"Cloud frontier for user chat, local for everything else"** — aligns with the `feedback_openclaw_local_background_routing.md` note. Pattern: Opus 4.7 or GPT-5.4 for the user-facing chat seat; Ollama-served Qwen 3.5 8B or Gemma 4 for heartbeats, crons, embeddings, classifiers.
2. **"OpenRouter `:floor` for classification"** — sub-penny per-classification at scale. Pair with a quality guard (cheap model does first pass, anything with low confidence escalates to mid-tier).
3. **"Qwen3-Coder-Next for autonomous code agents"** — r/LocalLLaMA consensus (per Latent.Space preview) for local coding agents that need to read files, apply patches, interpret test logs.
4. **"Multi-model cascade"** — aggregator recommendation: Gemma 4 E4B (free, on-device) for simple queries → Mistral Small 4 (low cost) for complex reasoning → Llama 4 Scout (10M context) for long-context retrieval.
5. **"MiMo-V2-Pro or DeepSeek V3.2 for bulk"** — common OpenRouter pattern for batch synthesis.
6. **"Claude Sonnet 4.6 for complex reasoning, Gemini Flash Lite for latency-sensitive"** — TeamDay.ai / aggregator pattern.
7. **"GLM-4.7-Flash as the local GPT-4o-mini"** — community framing (DeepReviewAI). Same niche, different sovereignty.
8. **"Kimi K2.5 for tool-chains beyond 50 calls"** — K2.5's 200-300 sequential tool call stability is the specific brag; pick it when an agent needs long unbroken tool sessions.
9. **"Qwen3-Embedding-8B over OpenAI text-embedding-3"** — MTEB leader and free-to-self-host. Default embedder of the open-source stack.

---

## Anti-recommendations (what NOT to use anymore)

- **MiniMax abab5.5** — superseded; the M2.x series is the current generation. Anything below M2.1 is legacy.
- **Kimi K2-Instruct (original)** — superseded by K2-Thinking (Nov 2025) and then K2.5 (Jan 2026). Kimi K3 not yet released as of April 2026.
- **Qwen 2.5-Coder 32B** — still works, but Qwen3-Coder-Next or Qwen3-Coder Plus is 2026's answer. A lot of blog posts still point to 2.5-Coder; those are stale.
- **DeepSeek V3 (non-.2)** — V3.2 and V4 both exist; V3 is legacy at this point.
- **GLM-4.6 and below** — 4.7 Flash / GLM-5 are current. 4.6 survives in some docs but is not the community pick.
- **Llama 3.3 / 3.4** — Llama 4 Scout/Maverick is where the Meta story is in 2026.
- **GPT-4o / GPT-4o-mini** — superseded by GPT-5.4 family; GPT-5.4 nano at $0.20/$1.25 replaces mini for classification budgets.
- **Claude 3.5 Sonnet / Haiku 3.5** — superseded by 4.6 / 4.5 respectively.
- **Gemini 2.5 Pro** in "latest flagship" claims — Gemini 3.1 Pro is the current top of that family per April 2026 benchmarks.
- **o1 / o1-mini** — DeepSeek R1 / R2 are ~96% cheaper on output tokens with comparable reasoning quality. Only pick o1 for OpenAI-ecosystem lock-in reasons.
- **Raw `openrouter/auto` without a cost guardrail** — community warning: auto is great for variable workloads, but for high-volume single-task pipelines, pinning to `model:floor` gives predictable cost.

---

## Citations (full URLs)

**Frontier tier:**
- https://lmcouncil.ai/benchmarks
- https://www.aipricing.guru/blog/claude-opus-4-7-vs-gpt-5-4-vs-gemini-3/
- https://www.datacamp.com/blog/opus-4-7-vs-gpt-5-4
- https://www.vellum.ai/blog/claude-opus-4-7-benchmarks-explained
- https://www.buildfastwithai.com/blogs/best-ai-models-april-2026
- https://x.com/ArtificialAnlys/status/1943166841150644622 (Grok 4 Intelligence Index 73)
- https://x.ai/news/grok-4
- https://x.ai/news/grok-4-fast

**MiniMax:**
- https://www.minimax.io/news/minimax-m25
- https://www.minimax.io/news/minimax-m27-en
- https://github.com/MiniMax-AI/MiniMax-M2
- https://artificialanalysis.ai/articles/minimax-m2-benchmarks-and-analysis
- https://wavespeed.ai/blog/posts/minimax-m2-7-self-evolving-agent-model-features-benchmarks-2026/
- https://thomas-wiegold.com/blog/minimax-m25-review/

**Kimi:**
- https://huggingface.co/moonshotai/Kimi-K2-Thinking
- https://huggingface.co/moonshotai/Kimi-K2.5
- https://www.infoq.com/news/2026/02/kimi-k25-swarm/
- https://platform.moonshot.ai/
- https://intuitionlabs.ai/articles/kimi-k2-technical-deep-dive
- https://medium.com/@mlabonne/kimi-k2-5-still-worth-it-after-two-weeks-f32abd991e26

**Qwen:**
- https://qwen.ai/blog?id=qwen3-coder-next
- https://huggingface.co/Qwen/Qwen3-Coder-Next
- https://tokenmix.ai/blog/qwen3-coder-guide
- https://eval.16x.engineer/blog/qwen3-coder-evaluation-results
- https://unsloth.ai/docs/models/qwen3.5
- https://qwenlm.github.io/blog/qwen3-embedding/
- https://huggingface.co/Qwen/Qwen3-Embedding-8B
- https://awesomeagents.ai/leaderboards/embedding-model-leaderboard-mteb-march-2026/
- https://dev.to/sienna/qwen3-coder-next-the-complete-2026-guide-to-running-powerful-ai-coding-agents-locally-1k95
- https://techie007.substack.com/p/qwen-35-the-complete-guide-benchmarks

**GLM:**
- https://huggingface.co/zai-org/GLM-4.7-Flash
- https://www.marktechpost.com/2026/01/20/zhipu-ai-releases-glm-4-7-flash-a-30b-a3b-moe-model-for-efficient-local-coding-and-agents/
- https://docs.z.ai/guides/llm/glm-4.7
- https://llm-stats.com/models/compare/glm-5-vs-glm-4.7-flash
- https://llm-stats.com/blog/research/glm-4.7-flash-launch
- https://deepreviewai.com/reviews/2026-02-05_glm47-flash-review/
- https://medium.com/@able_wong/glm-5-dropped-before-i-could-finish-testing-glm-4-7-722a056877ff

**DeepSeek:**
- https://deepseek.ai/blog/deepseek-guide-2026
- https://www.nxcode.io/resources/news/deepseek-v4-release-specs-benchmarks-2026
- https://openrouter.ai/deepseek/deepseek-v3.2
- https://www.nxcode.io/resources/news/deepseek-api-pricing-complete-guide-2026
- https://pricepertoken.com/pricing-page/provider/deepseek

**Local / open-weights aggregators:**
- https://www.latent.space/p/ainews-top-local-models-list-april
- https://www.maniac.ai/blog/chinese-frontier-models-compared-glm5-minimax-kimi-qwen
- https://benchlm.ai/blog/posts/best-chinese-llm
- https://onyx.app/open-llm-leaderboard
- https://localaimaster.com/models/best-local-ai-coding-models
- https://localaimaster.com/blog/small-language-models-guide-2026
- https://neural-digest.com/best-local-llm-to-run-2026-complete-guide/
- https://www.digitalapplied.com/blog/open-source-ai-landscape-april-2026-gemma-qwen-llama

**Gemma / Mistral / Llama:**
- https://blogs.nvidia.com/blog/rtx-ai-garage-open-models-google-gemma-4/
- https://www.interconnects.ai/p/gemma-4-and-what-makes-an-open-model
- https://unsloth.ai/docs/models/gemma-4
- https://ollama.com/library/gemma4
- https://mistral.ai/news/mistral-small-4
- https://huggingface.co/mistralai/Mistral-Small-4-119B-2603
- https://openrouter.ai/mistralai/mistral-small-2603
- https://www.mindstudio.ai/blog/what-is-mistral-small-4

**OpenRouter:**
- https://openrouter.ai/docs/guides/routing/provider-selection
- https://openrouter.ai/docs/guides/routing/model-variants/nitro
- https://openrouter.ai/announcements/introducing-nitro-and-floor-price-shortcuts
- https://openrouter.ai/docs/guides/routing/routers/auto-router
- https://openrouter.ai/docs/faq
- https://openrouter.ai/collections/free-models
- https://openrouter.ai/openrouter/free
- https://www.digitalapplied.com/blog/openrouter-rankings-april-2026-top-ai-models-data
- https://www.teamday.ai/blog/top-ai-models-openrouter-2026
- https://sidsaladi.substack.com/p/openrouter-101-the-complete-guide

**OpenClaw + OpenRouter:**
- https://openrouter.ai/docs/guides/coding-agents/openclaw-integration
- https://openrouter.ai/works-with-openrouter/moltbot
- https://openrouter.ai/apps/openclaw
- https://skywork.ai/skypage/en/openclaw-openrouter-configuration/2037009607672283136
- https://www.verdent.ai/guides/openclaw-setup-guide-from-zero-to-ai-assistant
- https://composio.dev/toolkits/openrouter/framework/openclaw
- https://github.com/openclaw/openclaw/releases/tag/v2026.4.7
- https://github.com/BlockRunAI/ClawRouter

**Anthropic / OpenAI / Google pricing:**
- https://platform.claude.com/docs/en/about-claude/pricing
- https://devtk.ai/en/blog/claude-api-pricing-guide-2026/
- https://www.digitalapplied.com/blog/claude-sonnet-4-6-benchmarks-pricing-guide
- https://aicostcheck.com/blog/gpt-5-4-mini-nano-pricing-benchmarks
- https://www.buildfastwithai.com/blogs/gpt-5-4-mini-nano-explained
- https://docsbot.ai/models/compare/gpt-5-4-nano/gemini-2.5-flash-lite
- https://sidmehtamit.medium.com/gpt-5-nano-vs-gemini-2-5-flash-lite-an-evaluation-of-cost-effective-ai-9e007b964a58

**Artificial Analysis comparisons:**
- https://artificialanalysis.ai/models/comparisons/minimax-m2-7-vs-kimi-k2-5
- https://artificialanalysis.ai/models/comparisons/glm-5-vs-kimi-k2-5
- https://artificialanalysis.ai/models/comparisons/glm-5-vs-minimax-m2-7
- https://artificialanalysis.ai/models/comparisons/kimi-k2-5-vs-minimax-m2
