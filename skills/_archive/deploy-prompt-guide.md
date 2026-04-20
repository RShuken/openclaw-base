# Deploy Prompt Engineering Guide

## Purpose

Install a prompt engineering reference document tailored to the client's primary AI model. Prevents common prompting mistakes that cause overtriggering, anti-pattern learning, and poor output formatting. The guide covers five universal principles that apply to all models, plus model-specific tips for whichever model the client actually uses.

**When to use:** First-time setup, when switching primary models, or when the agent's prompts need tuning.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/docs/PROMPTING-GUIDE.md 2>/dev/null"
cmd --session <id> --command "openclaw config get models.primary"
cmd --session <id> --command "openclaw config get models.list 2>/dev/null"
```
- Does a prompting guide already exist? If so, back up before overwriting.
- What is the client's primary model? This determines which model-specific section is included.
- Are multiple models in use? If so, include all relevant model-specific sections.

## Adaptation Points

The recommended defaults below come from the reference implementation (Matt Berman's setup using Claude Opus 4.6, battle-tested across 26 systems). Clients can use them as-is or customize any component.

| Component | Recommended Default | Customize When... |
|-----------|-------------------|-------------------|
| Target model | Claude Opus 4.6 | Client uses a different primary model (DeepSeek, GPT, Gemini, etc.) |
| Guide filename | `PROMPTING-GUIDE.md` | Client wants a different name or has an existing guide to merge with |
| Guide location | `${WORKSPACE}/docs/` | Non-standard workspace layout |
| Model-specific sections | Include only the primary model's section | Client uses multiple models and wants tips for all of them |
| Config reference | AGENTS.md + openclaw.json | Client uses a different config mechanism |

## Prerequisites

- OpenClaw agent installed
- Write access to `${WORKSPACE}/docs/`

## What Gets Installed

| File | Location | Purpose |
|------|----------|---------|
| `PROMPTING-GUIDE.md` | `${WORKSPACE}/docs/` | Universal prompting principles + model-specific tips |

## Steps

### 1. Check for Existing Guide

```
cmd --session <id> --command "ls ${WORKSPACE}/docs/PROMPTING-GUIDE.md ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md 2>/dev/null"
```

If a guide exists, back it up:

```
cmd --session <id> --command "cp ${WORKSPACE}/docs/PROMPTING-GUIDE.md ${WORKSPACE}/docs/PROMPTING-GUIDE.md.bak 2>/dev/null; cp ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md ${WORKSPACE}/docs/OPUS-PROMPTING-GUIDE.md.bak 2>/dev/null"
```

If migrating from an older `OPUS-PROMPTING-GUIDE.md`, the new file replaces it. The backup preserves any custom additions the client may have made.

### 2. Identify the Client's Primary Model

Check the client profile for their primary model. This determines which model-specific section to include.

```
cmd --session <id> --command "openclaw config get models.primary"
```

| Primary Model | Include Section |
|---------------|-----------------|
| Claude Opus 4.6 (default) | Claude Opus 4.6 tips |
| DeepSeek V3.x | DeepSeek tips |
| GPT-4.x / GPT-5.x | GPT tips |
| Gemini 2.x / Gemini 3.x | Gemini tips |
| Other / unknown | Generic tips |

Ask the client:
> "Your prompting guide needs to know your primary model. The default guide is optimized for Claude Opus 4.6. What model does your agent use day-to-day? If you use multiple models, I can include tips for all of them."

If the client uses multiple models (e.g., Opus for reasoning, DeepSeek for code), include all relevant sections.

### 3. Create the Prompting Guide

Create the docs directory if needed:

```
cmd --session <id> --command "mkdir -p ${WORKSPACE}/docs"
```

Write the guide. The **Universal Principles** and **Applying These Principles** sections are always included. The **Model-Specific Tips** section includes only the relevant model(s).

**Recommended template** (include the universal sections plus the model-specific section for the client's primary model):

```
cmd --session <id> --command "cat > ${WORKSPACE}/docs/PROMPTING-GUIDE.md << 'GUIDE_EOF'
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

### Claude Opus 4.6

Opus 4.6 is the most sensitive current model to prompting style. The universal principles above were originally discovered through production debugging on Opus.

- **Extremely sensitive to ALL-CAPS urgency.** Opus takes every capitalized instruction as a hard constraint and tries to honor all of them simultaneously. Even a handful of CRITICAL/MUST/NEVER markers causes noticeable overtriggering.
- **Excellent at generalizing from explanations.** Principle 2 (Explain the Why) is especially effective. Opus extrapolates "why" reasoning to edge cases the prompt never explicitly covers.
- **Will reproduce anti-patterns if shown them.** Even when clearly labeled "do not do this," Opus latches onto detailed anti-pattern examples. This is the model where Principle 3 matters the most. Show only desired behavior.
- **Mirrors prompt formatting faithfully.** Principle 5 is almost 1:1 with Opus. Whatever structure your prompt uses, Opus replicates it closely.
- **Doubt-based triggers cause massive overtriggering.** "If unsure" and "when in doubt" instructions cause Opus to fire tools on nearly every message. Always use specific trigger conditions.

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
GUIDE_EOF"
```

**Reference: Matt's version** (use as-is if client picks all defaults — Claude Opus 4.6 primary model):

```
cmd --session <id> --command "cat > ${WORKSPACE}/docs/PROMPTING-GUIDE.md << 'GUIDE_EOF'
# Prompting Guide

Best practices for writing prompts that work well with Claude Opus 4.6. Learned from real production usage and debugging sessions. The five universal principles apply to all modern language models. The model-specific section covers behaviors unique to Opus 4.6.

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

## Model-Specific Tips: Claude Opus 4.6

Opus 4.6 is the most sensitive current model to prompting style. The universal principles above were originally discovered through production debugging on Opus.

- **Extremely sensitive to ALL-CAPS urgency.** Opus takes every capitalized instruction as a hard constraint and tries to honor all of them simultaneously. Even a handful of CRITICAL/MUST/NEVER markers causes noticeable overtriggering.
- **Excellent at generalizing from explanations.** Principle 2 (Explain the Why) is especially effective. Opus extrapolates "why" reasoning to edge cases the prompt never explicitly covers.
- **Will reproduce anti-patterns if shown them.** Even when clearly labeled "do not do this," Opus latches onto detailed anti-pattern examples. This is the model where Principle 3 matters the most. Show only desired behavior.
- **Mirrors prompt formatting faithfully.** Principle 5 is almost 1:1 with Opus. Whatever structure your prompt uses, Opus replicates it closely.
- **Doubt-based triggers cause massive overtriggering.** "If unsure" and "when in doubt" instructions cause Opus to fire tools on nearly every message. Always use specific trigger conditions.

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
GUIDE_EOF"
```

### 4. Reference the Guide in Agent Config

Add a reference so the agent consults the prompting guide before writing or editing prompts.

**Option A: openclaw.json config**

```
cmd --session <id> --command "openclaw config set docs.prompt-guide ${WORKSPACE}/docs/PROMPTING-GUIDE.md"
```

**Option B: AGENTS.md reference** (if using an AGENTS.md file)

```
cmd --session <id> --command "cat >> ${WORKSPACE}/AGENTS.md << 'EOF'

## Prompt Engineering

Before writing or editing any prompt, system instruction, or tool description, consult `docs/PROMPTING-GUIDE.md`. Apply all five universal principles. Pay special attention to removing ALL-CAPS urgency markers and doubt-based tool triggers.
EOF"
```

If migrating from an older guide, clean up the old reference:

```
cmd --session <id> --command "sed -i.bak 's/OPUS-PROMPTING-GUIDE/PROMPTING-GUIDE/g' ${WORKSPACE}/AGENTS.md 2>/dev/null"
cmd --session <id> --command "openclaw config set docs.prompt-guide ${WORKSPACE}/docs/PROMPTING-GUIDE.md 2>/dev/null"
```

### 5. Verify

**File check:**

```
cmd --session <id> --command "test -f ${WORKSPACE}/docs/PROMPTING-GUIDE.md && echo 'Guide installed' || echo 'Guide MISSING'"
```

**Content check:**

```
cmd --session <id> --command "head -5 ${WORKSPACE}/docs/PROMPTING-GUIDE.md"
```

Expected output should show the title and introduction with the correct model name.

**Reference check:**

```
cmd --session <id> --command "grep -l 'PROMPTING-GUIDE' ${WORKSPACE}/AGENTS.md ${WORKSPACE}/openclaw.json 2>/dev/null"
```

At least one config file should reference the guide.

**Behavior test:**

Ask the agent to write a prompt for a new tool. The output should:
- Use normal case, not ALL-CAPS urgency markers
- Include reasoning for each rule
- Show only positive examples (no anti-patterns)
- Have specific tool trigger conditions, not "if in doubt" language
- Match the formatting style of the desired output

### 6. Update Client Profile

After deployment, update `clients/<name>.md` with:
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
| Agent still writes ALL-CAPS prompts | Guide not referenced in config | Add the reference to AGENTS.md or openclaw.json as shown in Step 4 |
| Guide exists but agent ignores it | Agent does not load docs/ files automatically | Add an explicit instruction in the system prompt to consult the guide before prompt work |
| Anti-patterns still appear in new prompts | Old prompts contain anti-patterns that the model copies | Audit existing prompts and clean them using the guide's principles |
| Tools still overtrigger after applying guide | Specific tool descriptions still have "if in doubt" language | Check each tool's description individually and remove vague triggers |
| Output formatting does not match desired style | System prompt style does not match desired output | Rewrite the system prompt in the target format per Principle 5 |
| Model-specific tips feel wrong | Wrong model section included, or model updated | Re-check the client's primary model and update the guide section |
| Old OPUS-PROMPTING-GUIDE.md still referenced | Migration from old guide incomplete | Run the sed cleanup in Step 4, update all config references |

## Dependencies

- **Depends on:** Nothing. Standalone reference document.
- **Required by:** No hard dependencies, but referenced by all systems that write or edit prompts.
- **Related:** `deploy-identity.md` (SOUL.md style rules align with prompting principles), `deploy-security-safety.md` (prompt injection defense benefits from clean prompting)
