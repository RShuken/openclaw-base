# Deploy Prompt Engineering Guide

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING
- **Approach**: writes `PROMPTING-GUIDE.md` to `~/.openclaw/workspace/docs/` which OpenClaw reads natively. File-based = version-proof.
- **Model identification**: use `openclaw models status` to see what the agent is currently routed to — that determines which Model-Specific Tips section to include. No need for a `docs.prompt-guide` config key.
- **Model**: agent default (Codex-first); the Model-Specific Tips section below covers Claude Opus 4.7, Codex/GPT-5.x, DeepSeek, Gemini, and Other.

## Purpose

Install a prompt engineering reference document tailored to the client's primary AI model. Prevents common prompting mistakes that cause overtriggering, anti-pattern learning, and poor output formatting. The guide covers five universal principles that apply to all models, plus model-specific tips for whichever model the client actually uses.

**When to use:** First-time setup, when switching primary models, or when the agent's prompts need tuning.

**What this skill does:**
1. Checks for an existing prompting guide and backs it up if found
2. Identifies the client's primary model (Claude, DeepSeek, GPT, Gemini, etc.)
3. Writes a `PROMPTING-GUIDE.md` with universal principles and model-specific tips
4. References the guide in the agent configuration so it is consulted automatically
5. Verifies the guide is installed and referenced

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${PRIMARY_MODEL}` | Client profile or `auth-profiles.json` provider field | `Claude Opus 4.6` |
| `${GUIDE_FILENAME}` | Default or client preference | `PROMPTING-GUIDE.md` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if a prompting guide already exists:

**Remote:**
```
ls ${WORKSPACE}/docs/PROMPTING-GUIDE.md ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md 2>/dev/null
```

Expected: Either "no such file" (clean install) or file path listed (existing guide to back up).

Check the client's primary model from auth-profiles.json:

**Remote:**
```
cat ${OPENCLAW_CONFIG}/agents/main/agent/auth-profiles.json 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print('\n'.join(f'{k}: {v.get(\"provider\",\"unknown\")}' for k,v in d.get('profiles',{}).items()))" 2>/dev/null || echo 'NO_AUTH_PROFILES'
```

Expected: List of provider profiles (e.g., `default: anthropic`, `google:default: google`). The primary model provider is the `default` profile. If no auth profiles exist, ask the client.

**Alternative:** Check the client profile for their primary model, or ask them directly.

**Decision points from pre-flight:**
- Does a prompting guide already exist? If so, back it up before overwriting.
- What is the client's primary model? This determines which model-specific section is included.
- Are multiple models in use? If so, include all relevant model-specific sections.

## Adaptation Points

The recommended defaults below are the OpenClaw-base reference starting point. Clients can use them as-is or customize any component.

| Component | Recommended Default | Customize When... |
|-----------|-------------------|-------------------|
| Target model | Model-agnostic by default; show tips for whichever model the agent is routed to | Client uses a different primary model (DeepSeek, GPT, Gemini, etc.) |
| Guide filename | `PROMPTING-GUIDE.md` | Client wants a different name or has an existing guide to merge with |
| Guide location | `${WORKSPACE}/docs/` | Non-standard workspace layout |
| Model-specific sections | Include only the primary model's section | Client uses multiple models and wants tips for all of them |
| Config reference | AGENTS.md + openclaw.json | Client uses a different config mechanism |

## Prerequisites

- OpenClaw agent installed (`openclaw/install.md`)
- Write access to `${WORKSPACE}/docs/`

## What Gets Installed

| File | Location | Purpose |
|------|----------|---------|
| `PROMPTING-GUIDE.md` | `${WORKSPACE}/docs/` | Universal prompting principles + model-specific tips |

## Steps

### Phase 1: Prepare

#### 1.1 Check for Existing Guide `[AUTO]`

Check whether a prompting guide already exists. If found, back it up so the client's custom additions are preserved.

**Remote:**
```
ls ${WORKSPACE}/docs/PROMPTING-GUIDE.md ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md 2>/dev/null
```

Expected: No output (clean install) or file paths listed.

If a guide exists, back it up:

**Remote:**
```
cp ${WORKSPACE}/docs/PROMPTING-GUIDE.md ${WORKSPACE}/docs/PROMPTING-GUIDE.md.bak 2>/dev/null; cp ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md.bak 2>/dev/null
```

Expected: Backup files created, or no-op if originals did not exist.

If migrating from an older `OPUS-PROMPTING-GUIDE.md`, the new file replaces it. The backup preserves any custom additions the client may have made.

If already exists with correct content: Skip the backup and proceed to verification. Note "guide already deployed."

#### 1.2 Identify the Client's Primary Model `[HUMAN_INPUT]`

Determine which model-specific section to include. Check the client profile first, then verify by reading auth-profiles.json.

**Remote:**
```
cat ${OPENCLAW_CONFIG}/agents/main/agent/auth-profiles.json 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); p=d.get('profiles',{}).get('default',{}); print(f'Provider: {p.get(\"provider\",\"unknown\")}')" 2>/dev/null || echo 'Cannot determine model'
```

Expected: Provider name (e.g., `Provider: anthropic`). Anthropic = Claude, google = Gemini, openai = GPT.

> **Note:** `openclaw config get models.primary` does not exist in OpenClaw 2026.4.15+ (bumped 2026-04-20). Determine the model from auth-profiles.json or ask the client directly.

| Primary Model | Include Section |
|---------------|-----------------|
| Claude Opus 4.7 (default) / Opus 4.6 (legacy) | Claude Opus 4.x tips |
| DeepSeek V3.x | DeepSeek tips |
| GPT-4.x / GPT-5.x | GPT tips |
| Gemini 2.x / Gemini 3.x | Gemini tips |
| Other / unknown | Generic tips |

Ask the client:
> "Your prompting guide needs to know your primary model. The default guide is optimized for Claude Opus 4.6. What model does your agent use day-to-day? If you use multiple models, I can include tips for all of them."

If the client uses multiple models (e.g., Opus for reasoning, DeepSeek for code), include all relevant sections.

If this fails: The model config may not be set yet. Ask the client directly or default to Claude Opus 4.6.

---

### Phase 2: Write the Prompting Guide

#### 2.1 Create the Docs Directory `[AUTO]`

Ensure the docs directory exists under the workspace.

**Remote:**
```
mkdir -p ${WORKSPACE}/docs
```

Expected: Directory exists at `${WORKSPACE}/docs/`.

If already exists: `mkdir -p` is idempotent. No action needed.

#### 2.2 Write the Guide File `[GUIDED]`

Write the prompting guide with the universal principles and the model-specific section(s) identified in Step 1.2.

The **Universal Principles** and **Applying These Principles** sections are always included. The **Model-Specific Tips** section includes only the relevant model(s).

**Remote:**
```
cat > ${WORKSPACE}/docs/PROMPTING-GUIDE.md << 'GUIDE_EOF'
# Prompting Guide

Best practices for writing prompts that work well with [MODEL_NAME]. Learned from real production usage and debugging sessions. The five universal principles apply to all modern language models. The model-specific section covers behaviors unique to the model this agent runs on.

---

## Universal Principles

These five principles apply to every modern language model. They are not model-specific. Apply all five to every prompt you write or edit.

### Principle 1: Drop the Urgency Theater

Older prompting advice said to use CAPS and strong words like CRITICAL, MUST, NEVER, ALWAYS to make the model pay attention. With current-generation models, this backfires.

**What happens:** The model treats every ALL-CAPS instruction as a hard constraint. When you have 10 "CRITICAL" rules, the model spends its reasoning budget trying to satisfy all of them simultaneously, even when they conflict. Result: overtriggering, excessive hedging, refusal to act.

**Instead:** Write rules in normal case. Use structure (headers, numbered lists) to indicate priority. Reserve strong language for genuinely dangerous operations (deleting production data, sending money).

The right way:
- Check the database when the user asks about stored data
- Include a disclaimer when giving medical or legal information
- Use the search tool when the answer requires current information you do not have

### Principle 2: Explain the Why

A rule with reasoning gets applied intelligently. A rule without reasoning gets applied mechanically.

Instead of: "Do not use the email tool for internal messages"

Write: "Do not use the email tool for internal messages. Internal messages go through Slack because the team checks Slack in real-time but only checks email twice a day. Using email for internal messages causes delays."

The "why" lets the model generalize. If someone asks to send an urgent internal message, the model with the "why" knows Slack is better because of the real-time checking. The model without the "why" might refuse to help entirely because the rule said "do not use email" and it has no basis for choosing an alternative.

### Principle 3: Show Only Desired Behavior

When you include anti-patterns (even labeled as "do not do this"), the model sometimes reproduces them. This is especially true when the anti-pattern is detailed or well-formatted.

Instead of showing what you do not want, show what you do want:
- Respond directly. Start with the answer. Example: "The config file is at ~/.openclaw/openclaw.json"

Only show the behavior you want to see. Every example in your prompt should be a positive example.

### Principle 4: Remove Doubt-Based Tool Triggers

Instructions like "if in doubt, use the search tool" or "when unsure, check the database" cause massive overtriggering. The model interprets almost any ambiguity as doubt.

Instead, be specific about when to use tools. Define the trigger conditions precisely:
- Use the search tool when the question is about events after your training cutoff, or when the user asks for specific numbers you do not have memorized

"Not sure" is never a good trigger because the model is "not sure" about a lot of things.

### Principle 5: Format Breeds Format

The model mirrors the style of the prompt. This is one of the most reliable behaviors.

- If your prompt is verbose, your output will be verbose
- If your prompt uses bullet points, your output will use bullet points
- If your prompt is concise and direct, your output will be concise and direct
- If your prompt uses headers and tables, your output will use headers and tables

Use this intentionally. Write your system prompt in the style you want the output to be.

---

## Model-Specific Tips

[Include the section(s) matching the client's model(s). Remove unneeded sections.]

### Claude Opus 4.7 (current flagship) + Opus 4.6 (legacy)

Opus 4.7 is the current Anthropic flagship (defaulted in OpenClaw 2026.4.15+); Opus 4.6 remains active as a legacy option. Opus is the most sensitive current model family to prompting style. The universal principles above were originally discovered through production debugging on the Opus 4.x family and apply to both 4.6 and 4.7.

- **Extremely sensitive to ALL-CAPS urgency.** Opus takes every capitalized instruction as a hard constraint and tries to honor all of them simultaneously. Even a handful of CRITICAL/MUST/NEVER markers causes noticeable overtriggering.
- **Excellent at generalizing from explanations.** Principle 2 (Explain the Why) is especially effective. Opus extrapolates "why" reasoning to edge cases the prompt never explicitly covers.
- **Will reproduce anti-patterns if shown them.** Even when clearly labeled "do not do this," Opus latches onto detailed anti-pattern examples. This is the model where Principle 3 matters the most. Show only desired behavior.
- **Mirrors prompt formatting faithfully.** Principle 5 is almost 1:1 with Opus. Whatever structure your prompt uses, Opus replicates it closely.
- **Doubt-based triggers cause massive overtriggering.** "If unsure" and "when in doubt" instructions cause Opus to fire tools on nearly every message. Always use specific trigger conditions.
- **Adaptive thinking only.** Opus 4.7 uses adaptive thinking (not extended thinking like Sonnet 4.6 or Haiku 4.5). When the skill needs deep reasoning, use `reasoning.effort` (none/low/medium/high/**xhigh**) — the `xhigh` setting was added in OpenClaw 2026.4.18 per baseline §8.
- **Opus 4.7 uses a new tokenizer** — 1M tokens ≈ 555k words (Sonnet 4.6 is ~750k words per 1M). Token budgets ported from 4.6 may miscalibrate.

### DeepSeek V3.x

DeepSeek V3 is a strong reasoning model with a code-first personality. The universal principles apply, with these model-specific notes:

- **Less sensitive to ALL-CAPS than Claude, but still benefits from removing them.** DeepSeek does not overtrigger as aggressively from urgency markers, but clean prompts still produce better results.
- **Benefits from chain-of-thought prompting.** When you need DeepSeek to reason through a problem, explicitly ask for step-by-step thinking. DeepSeek's reasoning improves noticeably when given permission to "think out loud."
- **Responds well to structured prompts with clear sections.** Use headers, numbered lists, and clear delimiters between sections. DeepSeek parses structure reliably.
- **Code-first examples work best.** When demonstrating desired behavior, use code snippets or structured data examples rather than prose descriptions. DeepSeek's strongest mode is code and structured output.
- **Benefits from explicit output format specification.** Tell DeepSeek exactly what format you want back (JSON, markdown table, bullet list, etc.). It follows format instructions more precisely when they are explicit rather than implied by prompt style.

### GPT Models (4.x, 5.x)

GPT models are widely used and well-understood. The universal principles apply, with these notes:

- **Moderately sensitive to urgency markers.** Less aggressive overtriggering than Claude, but ALL-CAPS still distort priority weighting. Remove them for cleaner results.
- **System message structure matters.** GPT models respond well to clear separation between system instructions and user messages. Put rules and constraints in the system message, examples and context in the user message.
- **Good at following structured output formats.** JSON mode and structured outputs work reliably. Use them when you need consistent formatting.
- **Less prone to anti-pattern reproduction.** GPT is less likely than Claude to latch onto "do not do this" examples, but Principle 3 is still good practice because it keeps prompts cleaner.
- **Function/tool calling works best with clear parameter descriptions.** When defining tools, write precise parameter descriptions with types and examples. Vague descriptions cause GPT to guess at parameter values.

### Gemini Models (2.x, 3.x)

Gemini has strong multimodal capabilities and structured output support. The universal principles apply, with these notes:

- **Benefits from multimodal examples when applicable.** If the task involves images, video, or audio, include relevant examples in the prompt rather than describing them in text.
- **Structured prompts with clear sections work well.** Like DeepSeek, Gemini parses headers and numbered lists reliably. Use structure to organize complex instructions.
- **Less sensitive to formatting mirroring than Claude.** Principle 5 still matters, but Gemini does not replicate prompt style as precisely as Opus. Be more explicit about desired output format.
- **Good at following specific output schemas.** When using Gemini's structured output mode, provide a precise schema. Gemini follows schemas more reliably than free-form format instructions.

### Other Models

If your primary model is not listed above, start with the five universal principles. They apply to all modern language models. Then test and iterate:

1. Write a prompt using all five principles
2. Run 5 representative queries
3. Check for overtriggering, anti-pattern reproduction, and formatting mismatches
4. Adjust based on what you observe
5. Document any model-specific discoveries below for future reference

---

## Applying These Principles

When writing or editing any prompt in the system:

1. Read through all instructions and remove ALL-CAPS urgency markers
2. For each rule, add a one-sentence explanation of why it exists
3. Remove all anti-pattern examples, replace with positive examples only
4. Replace vague tool triggers ("if unsure", "when in doubt") with specific conditions
5. Format the prompt in the style you want the output to match
6. Test with 5 representative queries and check for overtriggering

## Common Overtriggering Patterns

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Tool fires on every message | "If in doubt, use tool" instruction | Replace with specific trigger conditions |
| Model refuses simple requests | Too many NEVER/ALWAYS rules | Soften language, add context for when rules apply |
| Output is excessively cautious | Multiple CRITICAL warnings | Remove urgency markers, use normal priority language |
| Model includes unnecessary disclaimers | "ALWAYS include disclaimer" rule | Narrow the disclaimer rule to specific topics (medical, legal, financial) |
| Output is too long | System prompt is too long and verbose | Trim the system prompt, be concise |
| Model copies example formatting poorly | Anti-pattern examples in prompt | Remove anti-patterns, show only desired behavior |
GUIDE_EOF
```

> **Important:** Replace `[MODEL_NAME]` in the first paragraph with the client's actual primary model name before sending this command. If including only a subset of model-specific sections, remove the unneeded sections from the heredoc content before sending.

> **Important:** This guide exceeds 5.5KB. The 8KB session command limit applies. If the command fails due to payload size, split the write into two parts: write the universal principles first, then append the model-specific sections. Alternatively, use base64 encoding with chunked writes (see `e2e/lib/test-helpers.ts:writeFileCommands()` for the pattern).

**Reference: the default openclaw-base version** (use as-is if client picks all defaults):

The template above with `[MODEL_NAME]` replaced with whatever model `openclaw models status` reports as the active default (Codex GPT-5.4 for a fresh openclaw-base install per `templates/openclaw.json`). Only the matching Model-Specific Tips section needs to be retained in the generated file; the multi-model template is for clients who use multiple models.

Expected: File exists at `${WORKSPACE}/docs/PROMPTING-GUIDE.md` with the title "# Prompting Guide" and the correct model name in the introduction.

If this fails: Check that `${WORKSPACE}/docs/` directory exists (Step 2.1). Check the command payload size -- if it exceeds 8KB, split into chunks.

If already exists with correct content: Skip. Note "guide already deployed with correct content."

---

### Phase 3: Configure Agent Reference

#### 3.1 Reference the Guide in AGENTS.md `[GUIDED]`

Add a reference so the agent consults the prompting guide before writing or editing prompts.

> **Note:** `openclaw config set docs.prompt-guide` does not exist in OpenClaw 2026.4.15+ (bumped 2026-04-20). OpenClaw reads all `.md` files from the workspace directory as context. Writing the guide to `${WORKSPACE}/docs/` is sufficient -- no config registration is needed. However, an explicit reference in AGENTS.md reinforces the behavior.

First, check if AGENTS.md already references the guide:

**Remote:**
```
grep -c 'PROMPTING-GUIDE' ${WORKSPACE}/AGENTS.md 2>/dev/null || echo '0'
```

If the count is 0 (no existing reference), append the reference:

**Remote:**
```
cat >> ${WORKSPACE}/AGENTS.md << 'EOF'

## Prompt Engineering

Before writing or editing any prompt, system instruction, or tool description, consult `docs/PROMPTING-GUIDE.md`. Apply all five universal principles. Pay special attention to removing ALL-CAPS urgency markers and doubt-based tool triggers.
EOF
```

Expected: The AGENTS.md file contains a "Prompt Engineering" section referencing the guide.

If this fails: Check that `${WORKSPACE}/AGENTS.md` exists. If not, create it with this content as the initial section.

If already exists: If AGENTS.md already mentions `PROMPTING-GUIDE.md`, skip. If it references an old `OPUS-PROMPTING-GUIDE.md`, update the reference.

#### 3.2 Clean Up Old References `[AUTO]`

If migrating from an older `OPUS-PROMPTING-GUIDE.md`, update all references to point to the new filename.

**Remote:**
```
sed -i.bak 's/OPUS-PROMPTING-GUIDE/PROMPTING-GUIDE/g' ${WORKSPACE}/AGENTS.md 2>/dev/null
```

Expected: Any old references updated, or no-op if none existed.

> **Note:** No `openclaw config set` is needed here. OpenClaw reads `.md` files from the workspace by convention. The file at `${WORKSPACE}/docs/PROMPTING-GUIDE.md` is automatically available as context.

If this fails: Not critical. The old filename reference is cosmetic -- the new guide file is what matters.

If already migrated: Skip. No old references to clean up.

---

### Phase 4: Verify and Update Profile

#### 4.1 Verify Guide Installation `[AUTO]`

Confirm the guide file exists and has the correct content.

**File check:**

**Remote:**
```
test -f ${WORKSPACE}/docs/PROMPTING-GUIDE.md && echo 'Guide installed' || echo 'Guide MISSING'
```

Expected: `Guide installed`

**Content check:**

**Remote:**
```
head -5 ${WORKSPACE}/docs/PROMPTING-GUIDE.md
```

Expected: Title line `# Prompting Guide` and introduction with the correct model name.

**Reference check:**

**Remote:**
```
grep -l 'PROMPTING-GUIDE' ${WORKSPACE}/AGENTS.md 2>/dev/null
```

Expected: AGENTS.md listed as containing a reference to the guide. OpenClaw also reads the file directly from `${WORKSPACE}/docs/` by convention, so the AGENTS.md reference is reinforcement, not the only mechanism.

If this fails: Re-run Phase 3 to add the reference to AGENTS.md.

#### 4.2 Behavior Test `[GUIDED]`

Ask the agent to write a prompt for a new tool. The output should:
- Use normal case, not ALL-CAPS urgency markers
- Include reasoning for each rule
- Show only positive examples (no anti-patterns)
- Have specific tool trigger conditions, not "if in doubt" language
- Match the formatting style of the desired output

Expected: Agent-generated prompt follows all five universal principles without being reminded.

If this fails: Check that the guide reference (Step 3.1) is actually being loaded by the agent. Some agents require a restart to pick up new config references.

#### 4.3 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:
- Which primary model section was included (e.g., "Claude Opus 4.6" or "DeepSeek V3.2")
- Whether additional model sections were included
- Guide location (if non-default)
- Note: "Prompting Guide deployed on [date]"

## Verification

After deployment, confirm the guide is working:

1. **File check:** Guide exists at `${WORKSPACE}/docs/PROMPTING-GUIDE.md`
2. **Content check:** Read the file back, confirm universal principles and the correct model-specific section are present
3. **Reference check:** At least one config file (AGENTS.md or openclaw.json) references the guide
4. **Behavior check:** Start a new session, ask the agent to write a prompt. The agent should follow all five principles without being reminded.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Agent still writes ALL-CAPS prompts | Guide not referenced in config | Add the reference to AGENTS.md or openclaw.json (Phase 3, Step 3.1) |
| Guide exists but agent ignores it | Agent does not load docs/ files automatically | Add an explicit instruction in the system prompt to consult the guide before prompt work |
| Anti-patterns still appear in new prompts | Old prompts contain anti-patterns that the model copies | Audit existing prompts and clean them using the guide's principles |
| Tools still overtrigger after applying guide | Specific tool descriptions still have "if in doubt" language | Check each tool's description individually and remove vague triggers |
| Output formatting does not match desired style | System prompt style does not match desired output | Rewrite the system prompt in the target format per Principle 5 |
| Model-specific tips feel wrong | Wrong model section included, or model updated | Re-check the client's primary model and update the guide section |
| Old OPUS-PROMPTING-GUIDE.md still referenced | Migration from old guide incomplete | Run the sed cleanup in Step 3.2, update all config references |
| Command rejected (payload too large) | Guide content exceeds 8KB session command limit | Split into two writes or use base64 chunked encoding |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Check for existing guide | Execute silently | Execute silently | Confirm before checking |
| 1.2 Identify primary model | Ask client (always requires input) | Ask client | Ask client |
| 2.1 Create docs directory | Execute silently | Execute silently | Confirm before creating |
| 2.2 Write the guide file | Execute silently | Confirm before writing; show which model sections included | Confirm; show full content preview |
| 3.1 Reference in agent config | Execute silently | Confirm which config method (A or B) | Confirm each config change |
| 3.2 Clean up old references | Execute silently | Execute silently | Confirm before modifying files |
| 4.1 Verify installation | Execute silently | Execute silently | Show each check result |
| 4.2 Behavior test | Skip (trust the file) | Run test, show results | Run test, discuss results |
| 4.3 Update client profile | Execute silently | Execute silently | Show changes, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed so the workspace and config mechanism exist)
- **Required by:** No hard dependencies, but referenced by all systems that write or edit prompts.
- **Enhanced by:** `deploy-identity.md` (SOUL.md style rules align with prompting principles), `deploy-security-safety.md` (prompt injection defense benefits from clean prompting)
