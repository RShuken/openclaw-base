---
name: "Deploy AI Writing Humanizer"
category: "integration"
subcategory: "content-gen"
third_party_service: "n/a (local LLM pass)"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "HUMANIZER_MODEL"
clawhub_alternative: "@biostartechnology/humanizer"
cost_tier: "api-usage"
privacy_tier: "hybrid"
requires:
  - "Workspace skills/ directory writable"
  - "SOUL.md present"
docs_urls:
  - "https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing"
  - "https://docs.openclaw.ai/tools/skills.md"
---

# Deploy AI Writing Humanizer

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: NEEDS AUDIT
- **Potentially Blocked Commands**: `openclaw skills list`, `openclaw chat`
- **Notes**: References `openclaw skills list` (which exists and works) and `openclaw chat` (needs verification). The humanizer is a workspace file-based skill that may work if OpenClaw reads skill files from the workspace directory. Audit the skill installation mechanism before deploying.
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`HUMANIZER_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to the pattern-stripping pass — a cheap model is usually fine.

## Purpose

Install a skill that strips 24+ AI writing patterns from text before sending. Based on Wikipedia's "Signs of AI writing" page. Catches obvious tells and structural patterns. Runs automatically on user-facing prose.

**When to use:** Any time the agent generates text that will be read by humans. Deploy early -- improves all written output.

**What this skill does:**
1. Installs the humanizer skill directory into the agent's skill system
2. Verifies the skill loads and is recognized by OpenClaw
3. Ensures SOUL.md references the humanizer as the style pass
4. Validates output quality against the 24+ pattern checklist
5. Runs the regression test suite to confirm no known patterns slip through

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the humanizer skill is already installed:

**Remote:**
```
ls ${WORKSPACE}/skills/humanizer/ 2>/dev/null || echo 'NOT_INSTALLED'
```

Expected: Either the directory contents (already installed) or `NOT_INSTALLED`.

Check if SOUL.md exists and references the humanizer:

**Remote:**
```
grep -i 'humanizer' ${WORKSPACE}/SOUL.md 2>/dev/null || echo 'NO_REFERENCE'
```

Expected: A line referencing the humanizer, or `NO_REFERENCE` if SOUL.md is missing or lacks the reference.

**Decision points from pre-flight:**
- Is the humanizer already installed? If so, check the version and decide if an update is needed.
- Is SOUL.md deployed? If not, the humanizer still works but won't be referenced as the style pass. Consider deploying `deploy-identity.md` first or adding the reference manually.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Pattern list | 24+ standard AI writing patterns | Client's writing style naturally uses some "AI patterns" (e.g., they actually like em dashes) |
| Auto-run | Automatically on user-facing prose | Client wants more control over when humanizer runs |
| Style source | SOUL.md as single source of truth | Client has their own brand voice guidelines |
| Regression tests | Standard test suite | Client notices recurring patterns specific to their content |

## Prerequisites

- OpenClaw agent installed with skill system (`openclaw/install.md`)
- No external dependencies

## What Gets Installed

### Humanizer Skill (`skills/humanizer/`)

| Setting | Value |
|---------|-------|
| Version | v2.1.1 |
| Invocation | Agent skill call (no CLI commands) |
| Trigger | Runs automatically on longer user-facing prose |

### 24+ Detected Patterns

| Category | Examples |
|----------|---------|
| Inflated symbolism | Overly metaphorical language, forced analogies |
| Promotional language | "Groundbreaking", "Revolutionary", "Game-changing" |
| Em dash overuse | Replacing with commas, periods, or colons |
| Rule of three | "Fast, efficient, and reliable" -- simplify to what matters |
| AI vocabulary | "Delve", "Leverage", "Utilize", "Robust", "Seamless" |
| Hedging language | "It's worth noting", "It should be mentioned" |
| Stock phrases | "At the end of the day", "Holding down the fort", "Deep dive" |
| Performed authenticity | "As an AI, I...", fake empathy, disclaimers |
| List padding | Unnecessary bullet points that add nothing |
| Connector overuse | "Furthermore", "Moreover", "Additionally" |
| Passive voice | When active voice is more natural |
| Opening summaries | Restating the question before answering |

### Regression Tests

Catch patterns that keep coming back despite the rules. The test suite runs against a corpus of known-bad outputs and verifies the humanizer strips them. New patterns are added as discovered.

### SOUL.md Integration

SOUL.md references the humanizer as the single source of truth for writing cleanup: "invoke the humanizer skill as your style pass and treat it as the single source of truth." The humanizer enforces SOUL.md style rules (no em dashes, no preambles, no hedging) at the skill level so every output is clean.

## Steps

### Phase 1: Install the Humanizer Skill

#### 1.1 Copy Skill Directory `[AUTO]`

Copy the humanizer skill directory into the agent's skill system. The source location depends on how the skill files are distributed -- typically from a staging directory, a git checkout, or the operator's local machine via SCP.

**Remote:**
```
cp -r /path/to/humanizer ${WORKSPACE}/skills/humanizer
```

Expected: Directory `${WORKSPACE}/skills/humanizer/` exists with pattern definitions, regression tests, and the skill entry point.

If this fails: Verify the source path exists. If the skill files are on the operator's machine rather than the client, use SCP or the session commands API to transfer them.

If already exists: Compare the version. If the installed version matches or is newer, skip. If older, back up the existing directory as `humanizer.bak` and copy the new version.

#### 1.2 Verify Directory Structure `[AUTO]`

Confirm the skill files were copied correctly.

**Remote:**
```
ls ${WORKSPACE}/skills/humanizer/
```

Expected: Pattern definitions, regression tests, and the skill entry point are all present.

If this fails: The copy in step 1.1 may have been incomplete. Re-copy the directory.

### Phase 2: Verify Skill Integration

#### 2.1 Confirm Skill Loads `[AUTO]`

Verify that the OpenClaw skill system recognizes the humanizer.

**Remote:**
```
openclaw skills list | grep humanizer
```

Expected: `humanizer` appears in the skill list with status `loaded`.

If this fails: The skill entry point may be malformed or missing required metadata. Check the skill directory for the expected entry point file. Restart the agent if the skill was installed while it was running.

#### 2.2 Verify SOUL.md Reference `[GUIDED]`

SOUL.md should reference the humanizer as the style pass. Check if the reference exists.

**Remote:**
```
grep -i 'humanizer' ${WORKSPACE}/SOUL.md
```

Expected: A line such as "invoke the humanizer skill as your style pass and treat it as the single source of truth for writing cleanup."

If this fails (no match or SOUL.md missing): Add the reference to SOUL.md's Style Rules section.

**Remote:**
```
printf '\n- Invoke the humanizer skill as your style pass and treat it as the single source of truth for writing cleanup.\n' >> ${WORKSPACE}/SOUL.md
```

Expected: The line is appended to SOUL.md.

If this fails: Check that `${WORKSPACE}/SOUL.md` exists. If SOUL.md has not been deployed yet, note this as a follow-up for `deploy-identity.md` -- the humanizer still works without the SOUL.md reference but won't be invoked automatically as a style pass.

If already exists: Skip. The reference is already in place.

### Phase 3: Validate Output Quality

#### 3.1 Test Output Quality `[GUIDED]`

Ask the agent to write a paragraph and verify it comes back clean of AI patterns.

**Remote:**
```
openclaw chat --message 'Write a paragraph about why Postgres is a good database choice.' --one-shot
```

To route the humanizer pattern-stripping pass through a specific model, prefix with the env var (and have the skill's invocation pipeline read it):

```
HUMANIZER_MODEL=openai-codex/gpt-5.4-nano openclaw chat --message '...' --one-shot
```

Reference pattern for any bash helper that wraps the humanizer:

```bash
MODEL_FLAG=""
if [ -n "${HUMANIZER_MODEL:-}" ]; then
  MODEL_FLAG="--model ${HUMANIZER_MODEL}"
fi
openclaw agent --agent main $MODEL_FLAG -m "<humanize this>"
```

Check the output manually for:
- No em dashes
- No "it's worth noting" or "at the end of the day"
- No "delve", "leverage", "utilize", "robust", "seamless"
- No "Furthermore", "Moreover", "Additionally"
- No restating the question before answering
- No "Certainly!" or "Great question!" preambles

Expected: Clean prose free of the 24+ detected patterns. The writing should sound natural and direct.

If this fails: The humanizer may not be triggering on this type of output. Check that the skill is loaded (step 2.1) and that the agent's configuration includes automatic humanizer invocation for user-facing prose.

#### 3.2 Run Regression Tests `[AUTO]`

Run the regression test suite to verify no known patterns slip through.

**Remote:**
```
cd ${WORKSPACE}/skills/humanizer && npm test
```

Expected: All regression tests pass.

If this fails: One or more previously-fixed patterns have regressed. The test output will identify the specific pattern. Fix the pattern definition and re-run tests.

## Verification

After deployment, run these checks:

Confirm skill is loaded:

**Remote:**
```
openclaw skills list | grep humanizer
```

Run regression tests:

**Remote:**
```
cd ${WORKSPACE}/skills/humanizer && npm test
```

Then manually test by asking the agent to write several types of content:
- A recommendation (should be direct, not hedged)
- A technical explanation (should avoid "utilize", "leverage", "robust")
- A paragraph (should have no em dashes, no connector overuse)
- A response to a question (should not restate the question first)

If any patterns slip through, add them to the regression test suite and update the pattern definitions.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Patterns still appearing in output | Humanizer skill not loaded or not invoked | Check `openclaw skills list` for humanizer status. Restart the agent if needed. |
| Over-correction (output sounds robotic) | Patterns too aggressive, stripping natural phrasing | Review the pattern definitions. The humanizer should preserve voice while removing tells. Adjust pattern sensitivity. |
| Regression on old patterns | New update reverted a fix | Run regression tests. If a test fails, the specific pattern that regressed will be identified. |
| Em dashes still present | Em dash pattern not matching Unicode variants | Check that the pattern catches all em dash characters: `---`, `--`, and the Unicode em dash `\u2014`. |
| SOUL.md style rules not enforced | Humanizer not referenced in SOUL.md | Add the reference per Step 2.2. The humanizer enforces SOUL.md rules at the skill level. |
| "Furthermore/Moreover" still appearing | Connector pattern not broad enough | Add the specific connectors to the pattern list. Common misses: "In addition", "On top of that", "What's more". |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Copy skill directory | Execute silently | Confirm before copying | Confirm command |
| 1.2 Verify directory structure | Execute silently | Execute silently | Confirm command |
| 2.1 Confirm skill loads | Execute silently | Execute silently | Confirm command |
| 2.2 Verify SOUL.md reference | Execute silently | Confirm before modifying SOUL.md | Confirm command |
| 3.1 Test output quality | Execute silently | Review output together | Review output together |
| 3.2 Run regression tests | Execute silently | Execute silently | Confirm command |

## Dependencies

- **Depends on:** OpenClaw skill system (`openclaw/install.md`)
- **Referenced by:** `deploy-identity.md` (SOUL.md references humanizer as style pass)
- **Independent of:** All other deploy skills. Can be deployed at any time, but earlier is better.
