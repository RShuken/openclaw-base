# Deploy AI Writing Humanizer

## Purpose

Install a skill that strips 24+ AI writing patterns from text before sending. Based on Wikipedia's "Signs of AI writing" page. Catches obvious tells and structural patterns. Runs automatically on user-facing prose.

**When to use:** Any time the agent generates text that will be read by humans. Deploy early -- improves all written output.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/skills/humanizer/ 2>/dev/null"
cmd --session <id> --command "ls ${WORKSPACE}/SOUL.md 2>/dev/null"
```
- Is the humanizer already installed? Check for existing version.
- Is SOUL.md deployed? (References the humanizer as the style pass)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Pattern list | 24+ standard AI writing patterns | Custom additions or removals | Client's writing style naturally uses some "AI patterns" (e.g., they actually like em dashes) |
| Auto-run | Automatically on user-facing prose | Manual invocation only | Client wants more control over when humanizer runs |
| Style source | SOUL.md as single source of truth | Custom style guide | Client has their own brand voice guidelines |
| Regression tests | Standard test suite | Extended with client-specific patterns | Client notices recurring patterns specific to their content |

## Prerequisites

- OpenClaw agent installed with skill system
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

### 1. Install the Humanizer Skill

Copy the humanizer skill directory into the agent's skill system.

```
cmd --session <id> --command "cp -r /path/to/humanizer ~/clawd/skills/humanizer"
```

Verify the directory structure:

```
cmd --session <id> --command "ls ~/clawd/skills/humanizer/"
```

Expected: pattern definitions, regression tests, and the skill entry point.

### 2. Verify Skill Loads

Confirm the skill system recognizes the humanizer.

```
cmd --session <id> --command "openclaw skills list | grep humanizer"
```

Expected: `humanizer` appears in the skill list with status `loaded`.

### 3. Verify SOUL.md Reference

SOUL.md should already reference the humanizer as the style pass. Confirm the reference exists.

```
cmd --session <id> --command "grep -i 'humanizer' ~/clawd/SOUL.md"
```

If missing, add the reference to SOUL.md's Style Rules section:

```
cmd --session <id> --command "echo '\n- Invoke the humanizer skill as your style pass and treat it as the single source of truth for writing cleanup.' >> ~/clawd/SOUL.md"
```

### 4. Test Output Quality

Ask the agent to write a paragraph and verify it comes back clean of AI patterns.

```
cmd --session <id> --command "openclaw chat --message 'Write a paragraph about why Postgres is a good database choice.' --one-shot"
```

Check the output manually for:
- No em dashes
- No "it's worth noting" or "at the end of the day"
- No "delve", "leverage", "utilize", "robust", "seamless"
- No "Furthermore", "Moreover", "Additionally"
- No restating the question before answering
- No "Certainly!" or "Great question!" preambles

### 5. Run Regression Tests

```
cmd --session <id> --command "cd ~/clawd/skills/humanizer && npm test"
```

All regression tests should pass. If any fail, the humanizer has regressed on a previously-fixed pattern.

## Verification

After deployment, run these checks:

```
cmd --session <id> --command "openclaw skills list | grep humanizer"
cmd --session <id> --command "cd ~/clawd/skills/humanizer && npm test"
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
| SOUL.md style rules not enforced | Humanizer not referenced in SOUL.md | Add the reference per Step 3. The humanizer enforces SOUL.md rules at the skill level. |
| "Furthermore/Moreover" still appearing | Connector pattern not broad enough | Add the specific connectors to the pattern list. Common misses: "In addition", "On top of that", "What's more". |

## Dependencies

- **Requires:** OpenClaw skill system
- **Referenced by:** `deploy-identity.md` (SOUL.md references humanizer as style pass)
- **Independent of:** All other deploy skills. Can be deployed at any time, but earlier is better.
