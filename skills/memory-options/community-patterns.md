---
name: "Community memory patterns"
category: "community-pattern"
repo: "N/A (documentation of patterns, not a tool)"
license: "N/A"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active-pattern"
openclaw_integration: "apply on top of any backend"
cost_tier: "N/A"
privacy_tier: "N/A"
requires:
  - "Any OpenClaw memory backend"
docs_urls:
  - "../../../audit/_baseline.md"
  - "https://velvetshark.com/openclaw-memory-masterclass"
  - "https://blog.dailydoseofds.com/p/openclaws-memory-is-broken-heres"
---

# Community memory patterns

Apply these patterns on top of any memory backend (built-in, Lossless Claw, mem0, Graphiti, whatever). They're additive, not replacement.

## Weekly rollup

Past ~400 files in `memory/`, retrieval quality collapses (baseline §K.352). Run a weekly script that summarizes the last week's `memory/YYYY-MM-DD.md` into a single rollup file, archives originals. Keeps `memory/` small and discriminative.

## Hierarchical folders

Instead of flat `memory/YYYY-MM-DD.md`, split by project/entity/topic: `memory/people/`, `memory/projects/X/`, `memory/decisions/`. Downstream cost: embedding pipeline must respect the folder structure. Trade file-path discipline for survivable scale.

## Explicit Hot / Warm / Cold tiering (the reference snapshot's {{AGENT_NAME}} pattern)

the reference snapshot's {{AGENT_NAME}} codifies the memory pile into three explicit tiers, giving operational teeth to the "keep the kernel small" rule:

| Tier | Where | Loaded when | Size budget |
|------|-------|-------------|-------------|
| **Hot** | `workspace/MEMORY.md` (flat root file) | Every turn | **<200 lines** |
| **Warm** | `workspace/memory/YYYY-MM-DD.md` daily notes | On-demand via vector search | Whatever heartbeat accumulates in a day |
| **Cold** | `workspace/memory/archive/<year>/<week>.md` rollups | Only via explicit `lcm_grep` / Graphiti / mem0 lookup | Unlimited — lossless archive |

Pairs well with weekly-rollup: every Monday, Warm entries older than 7 days get promoted into a Cold weekly rollup. Warm stays under ~400 files (below quality-collapse threshold from baseline §K.352).

## Kernel vs memory split

Flat-root `.md` files (`AGENTS`, `SOUL`, `IDENTITY`, `HEARTBEAT`, `MEMORY`, `TOOLS`, `USER`, `BOOTSTRAP`, `BOOT`) are always loaded — keep them under a stable token budget. `memory/YYYY-MM-DD.md` are vector-indexed and retrieved on demand.

## MEMORY.md security gate

**Never load `MEMORY.md` in group chats or sub-agents.** Boot sequence must enforce. Applies regardless of memory backend (baseline §§K.153, K.405, K.430).

## Identity split

`SOUL.md` (philosophy) + `IDENTITY.md` (name/voice/emoji) + `MEMORY.md` (index). Three files, three concerns (baseline §K.351).

## Entity extraction via plugin

Don't extract entities in pure markdown — delegate to a structured store (`mem0.md`, `graphiti-neo4j.md`, or Cognee).

## Heartbeat memory auto-save (the reference snapshot's {{AGENT_NAME}} pattern)

Make the heartbeat the thing that writes memory. Every ~30 minutes, a local-model-backed heartbeat:

1. Reads recent session activity (what the agent did since last heartbeat)
2. Extracts 2–5 salient facts worth remembering
3. Appends them to today's `memory/YYYY-MM-DD.md`
4. Invokes structured-memory layer (mem0 `memory_store` or Graphiti node/edge write) if entities were mentioned
5. Runs cheap Lossless Claw summary pass while it's already in context

Why this wins: memory gets written continuously, not at session-end (which often never fires on 24/7 agents). Critical that the heartbeat model is **local** (oMLX / Ollama tier) — cost has to round to zero per fire.

**Caveat:** OpenClaw's `agents.defaults.heartbeat.model` override is broken ([#58137](https://github.com/openclaw/openclaw/issues/58137)). If the bug is present on your target, call the local model explicitly via `openclaw agent --model omlx:gemma4 -m "..."` from a cron-driven script instead of relying on heartbeat config.

## Hybrid model routing for background writes

Primary (cloud) model for user-facing chats only; heartbeat / dreaming / embeddings run on local (`omlx:gemma4` or `ollama:gemma4`). Matches `feedback_openclaw_local_background_routing.md`. **Blocked on #58137 / #65218** — verify on target OpenClaw version.

## Known OpenClaw memory failure modes (2026)

| Issue | GitHub | Status | Symptom |
|-------|--------|--------|---------|
| `heartbeat.model` / `compaction.model` override ignored | [#58137](https://github.com/openclaw/openclaw/issues/58137) | closed | Forces compaction/heartbeat back to primary model |
| Compaction-reserve precheck forces 16384-token reserve | [#65218](https://github.com/openclaw/openclaw/issues/65218) | open (2026-04-11) | Precheck ignores configured `reserveTokens` |
| Dreaming pollutes daily files | [#66947](https://github.com/openclaw/openclaw/issues/66947) | open | Dreaming writes daily memory; heartbeat sees file non-empty and skips |
| Pre-compaction flush overwrites | [#44787](https://github.com/openclaw/openclaw/issues/44787) | open | Agent overwrites daily files instead of appending |
| Compaction → silent hallucinations | baseline §K.3 | design limit | Multi-step procedures flattened into lossy summaries |
| Memory folder ≥400 files → quality collapse | baseline §K.352 | design limit | Retrieval quality degrades; compaction loses discriminative context |
| Heartbeat prompt-injection persistence (adversarial) | [arxiv 2603.23064v2](https://arxiv.org/html/2603.23064v2) | research | Polluted heartbeat content persists across compaction with up to 75.6% attack success |

**Single best fix for the first four + the compaction hallucination mode:** install `lossless-claw.md` in the `contextEngine` slot.

## Citations

- `audit/_baseline.md` §K (community-reported memory issues)
- https://velvetshark.com/openclaw-memory-masterclass
- https://blog.dailydoseofds.com/p/openclaws-memory-is-broken-heres
- https://kaxo.io/insights/openclaw-production-gotchas/
