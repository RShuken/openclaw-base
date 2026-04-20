# Skill Authoring Guide

How to write and maintain intent-based deploy skills for OpenClaw.

## The Philosophy

Skills are **runbooks for an intelligent AI operator**, not shell scripts to copy-paste.

The operator (whichever AI model `openclaw.json` routes to — the openclaw-base default stack is OpenAI Codex GPT-5.4, see `audit/_baseline.md` §BB) is highly capable. It can:
- Adapt bash commands to PowerShell, fish, or any shell
- Figure out platform-specific package managers
- Handle edge cases you didn't anticipate
- Read error messages and troubleshoot

What it needs from skills:
- **What** to accomplish at each step (the intent)
- **Why** it matters (so it can make tradeoffs)
- **Where exact syntax is critical** (so it doesn't improvise when it shouldn't)
- **What success looks like** (so it knows when to move on)
- **What failure looks like** (so it can troubleshoot)

### The 3-Level Abstraction Model

| Level | Description | Use When | Example |
|-------|-------------|----------|---------|
| **Level 1: Exact syntax** | Provide the exact command, no adaptation | Service configs, daemon files, JSON schemas, security settings | `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.openclaw.plist` |
| **Level 2: Intent + reference** | Describe what to do, show a reference command | Most steps — the operator adapts to the platform | "Create the workspace directory. Reference: `mkdir -p ${WORKSPACE}/data`" |
| **Level 3: Intent + knowledge** | Describe the goal, let the operator figure out how | High-level orchestration, decision points | "Schedule a nightly sync at the client's preferred time" |

**Default to Level 2-3.** Only use Level 1 for sequences where exact syntax matters for correctness.

## Skill Anatomy

Every deploy skill follows this structure. Reference `openclaw/install.md` as the gold standard.

### 1. Title and Purpose

```markdown
# Deploy [System Name]

## Purpose

One paragraph: what this system does and why a client needs it.

**When to use:** Trigger conditions.

**What this skill does:**
1. High-level step summary
2. ...
```

### 2. How Commands Are Sent

Explain the two command paths. This is the same across all skills — copy from `_deploy-common.md` or reference it:

```markdown
## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.
```

### 3. Variables

Table of all variables used in this skill, with source and example:

```markdown
## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${TELEGRAM_TOPIC_ID}` | Created during messaging setup | `1207` |
```

**Rules:**
- Every variable used in the skill MUST appear in this table
- Include the source (where the operator gets this value)
- Include a realistic example
- Don't hardcode values that should be variables (e.g., topic IDs, project IDs)

### 4. Before You Start

Reference the Standard Deployment Protocol and add skill-specific pre-flight checks:

```markdown
## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**
- Check if [system] is already deployed: [how to check]
- Verify prerequisite: [what to verify]
```

### 5. Adaptation Points

Table of what can be customized:

```markdown
## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Database path | `${WORKSPACE}/data/system.db` | Client wants DBs in a different location |
```

### 6. Prerequisites

Bullet list of what must exist before this skill runs.

### 7. Steps (Phases)

The core of the skill. Each phase is a logical group of steps.

```markdown
## Steps

### Phase 1: [Phase Name]

#### 1.1 [Step Name]

[Intent description — what we're doing and why]

**Remote:**
```
[reference command]
```

Expected: [what success looks like]

If this fails: [what to check]

If already exists: [idempotency behavior]
```

### 8. Verification

Post-deployment checks to confirm everything works.

### 9. Autonomy Mode Behavior

Table showing how each step behaves in auto/guided/manual modes.

### 10. Dependencies

What this skill depends on and what depends on it.

## How to Write a Step

Every step follows this pattern:

### 1. Start with INTENT

Describe **what** you're doing and **why**, in plain language:

> "Write the AI prompt configuration file that controls how the agent responds.
> This defines the system prompt, temperature, and model selection."

### 2. Add REFERENCE COMMAND

Show a reference command (usually bash). Prefix with `Remote:` or `Operator:`:

```markdown
**Remote:**
```
cat > ${WORKSPACE}/config/prompt.json << 'EOF'
{...}
EOF
```
```

### 3. Add EXPECTED outcome

What does success look like?

```markdown
Expected: File exists at `${WORKSPACE}/config/prompt.json` with valid JSON content.
```

### 4. Add ERROR indicators

What are the common failure modes?

```markdown
If this fails: Check that `${WORKSPACE}/config/` directory exists. Create it first if missing.
```

### 5. Add IDEMPOTENCY note

What happens if this step runs again?

```markdown
If already exists: Compare content. If unchanged, skip. If different, back up existing file as `prompt.json.bak` and write new version.
```

### 6. Add AUTONOMY marker

```markdown
`[GUIDED]` — Confirm in guided/manual mode before writing config.
```

### Full Example

```markdown
#### 2.3 Write Prompt Configuration `[GUIDED]`

Write the AI prompt configuration that controls agent behavior.
This defines the system prompt, model selection, and response parameters.

**Remote:**
```
cat > ${WORKSPACE}/config/prompt.json << 'EOF'
{
  "_comment": "Defer to the agent's default model stack; do not hardcode a model ID here. Model deprecations happen monthly — see audit/_baseline.md §X.",
  "temperature": 0.7,
  "systemPrompt": "You are a helpful assistant."
}
EOF
```

Expected: File exists at `${WORKSPACE}/config/prompt.json` with valid JSON.

If this fails: Check that `${WORKSPACE}/config/` directory exists.

If already exists: Compare content. If unchanged, skip. If different, backup as `.bak` and write new.

> **Why no hardcoded model ID here?** Hardcoding a specific model ID (e.g., `claude-sonnet-4-20250514`) in a skill's generated config makes the skill a ticking bomb — the Anthropic/OpenAI retirement schedule will break every client install on a fixed date. Skills should read `agents.defaults.model` from `openclaw.json` at runtime, or pin to a stable alias (`claude-sonnet-4-6`, `gpt-5.3-codex`) only when a specific capability is required. Both approaches survive point-in-time snapshot retirements.
```

## Anti-Patterns

These are mistakes found across the original skills. Do NOT repeat them.

### 1. Fictional CLI Commands

**Bad:**
```
cmd --session <session-id> "mkdir -p ~/clawd/data"
```

There is no `cmd` CLI. Commands go through the session API or run on the operator's terminal.

**Good:**
```markdown
**Remote:**
```
mkdir -p ${WORKSPACE}/data
```
```

### 2. Hardcoded Paths

**Bad:**
```
cat > ~/clawd/scripts/backup.sh
```

**Good:**
```markdown
Write the backup script to `${WORKSPACE}/scripts/backup.sh`.

**Remote:**
```
cat > ${WORKSPACE}/scripts/backup.sh << 'EOF'
...
EOF
```
```

### 3. `echo '\n'` for Blank Lines

**Bad:** `echo '\n'` — This outputs the literal string `\n` in most shells (bash `echo` without `-e`).

**Good:** Use `printf '\n'` or `echo ""` or just add blank lines in heredocs.

### 4. Quoted Heredocs with Variables

**Bad:**
```bash
cat << 'SCRIPT_EOF'
WORKSPACE_DIR="${WORKSPACE}"
...
SCRIPT_EOF
```

Single-quoted heredoc delimiters (`'SCRIPT_EOF'`) prevent variable expansion. The file will contain the literal string `${WORKSPACE}`, not the value.

**Good — if variables should expand:**
```bash
cat << SCRIPT_EOF
WORKSPACE_DIR="${WORKSPACE}"
SCRIPT_EOF
```

**Good — if the file should contain literal `$` references:**
Describe the intent and note that the operator must substitute values before sending.

### 5. `echo -e` for Escape Sequences

**Bad:** `echo -e "line1\nline2"` — Not portable. On macOS with `/bin/sh`, `echo -e` outputs the literal `-e`.

**Good:** `printf "line1\nline2\n"` — Works everywhere.

### 6. `for file in $(command)` for Iteration

**Bad:** `for file in $(git diff --name-only); do ...` — Breaks on filenames with spaces.

**Good:** `git diff --name-only | while IFS= read -r file; do ...`

### 7. macOS-Only Package Managers

**Bad:** `brew install uv` — Only works on macOS.

**Good:** Describe the intent: "Install uv (Python package installer). Reference: `brew install uv` (macOS), `curl -LsSf https://astral.sh/uv/install.sh | sh` (Linux), `irm https://astral.sh/uv/install.ps1 | iex` (Windows)."

### 8. Hardcoded Topic/Project IDs

**Bad:** `Send to Telegram topic 1207`

**Good:** Use a variable: `Send to Telegram topic ${FOOD_JOURNAL_TOPIC_ID}`. Add it to the Variables table with source: "Created during messaging setup" or "Client profile: messaging config."

### 9. Orphaned Scripts

**Bad:** Creating a script file but never scheduling or calling it.

**Good:** Every script created must have a corresponding step that schedules, calls, or registers it.

### 10. `wc -c` on macOS

**Bad:** `SIZE=$(wc -c < file.txt)` then comparing `$SIZE` — `wc -c` on macOS has leading whitespace.

**Good:** `SIZE=$(wc -c < file.txt | tr -d ' ')` or `SIZE=$(stat -f%z file.txt)` (macOS) / `SIZE=$(stat -c%s file.txt)` (Linux).

## Cross-Platform Reference Table

Common bash commands and their platform equivalents. For the full table, see `_deploy-common.md` → "Platform Adaptation."

| Intent | bash | PowerShell | Notes |
|--------|------|------------|-------|
| Create directory | `mkdir -p dir` | `New-Item -ItemType Directory -Force dir` | |
| Write file | `cat > file << 'EOF'` | `Set-Content file '...'` | For emoji/unicode, use `[System.IO.File]::WriteAllText()` on Windows |
| Append to file | `>> file` | `Add-Content file '...'` | |
| Check if file exists | `test -f file` | `Test-Path file` | |
| Read file | `cat file` | `Get-Content file` | |
| Delete recursively | `rm -rf dir` | `Remove-Item -Recurse -Force dir` | |
| Current user | `whoami` | `$env:USERNAME` | |
| Home directory | `$HOME` | `$env:USERPROFILE` | |
| Schedule (cron) | `crontab -e` | `schtasks /create ...` | See `_deploy-common.md` for details |
| pip install | `python3 -m pip install` | `python -m pip install` | Always use `python3 -m pip`, not bare `pip` |
| npm global | `npm install -g pkg` | `npm install -g pkg` | Same syntax |

## New Skill Template

```markdown
# Deploy [System Name]

## Purpose

[What this system does and why a client needs it. 2-3 sentences.]

**When to use:** [Trigger conditions]

**What this skill does:**
1. [High-level step 1]
2. [High-level step 2]
3. [High-level step 3]

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**
- [Check 1]
- [Check 2]

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| [Component] | [Default] | [When to customize] |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- [Other prerequisites]

## Steps

### Phase 1: [Phase Name]

#### 1.1 [Step Name] `[AUTO]`

[Intent: what we're doing and why.]

**Remote:**
\```
[reference command]
\```

Expected: [what success looks like]

If this fails: [what to check]

## Verification

[Post-deployment checks]

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| [Step] | [Behavior] | [Behavior] | [Behavior] |

## Dependencies

- **Depends on:** [skills this requires]
- **Required by:** [skills that need this]
```

---

## Two skill formats

OpenClaw's ecosystem has two different things both called "skills," and they're not interchangeable. Pick the right one for what you're building.

### Our `deploy-X.md` runbook format (this repo)

- Human-readable operator runbook — describes intent + commands for an operator (human or agent) to execute
- Lives at `skills/deploy-*.md` in `openclaw-base`
- Targets a fresh OpenClaw install and installs a system into the workspace
- Follows the structure defined above: Compatibility → Purpose → Variables → Before You Start → Adaptation Points → Prerequisites → What Gets Installed → Steps → Verification → Troubleshooting → Autonomy → Dependencies
- Markdown-only; no YAML frontmatter
- NOT directly loadable by the agent at runtime — these are deployment instructions, not agent context

### Canonical SKILL.md (loaded by the agent; publishable to ClawHub)

- Agent-loadable skill with YAML frontmatter + markdown body
- Lives in a **directory** like `skills/<author>/<slug>/SKILL.md` (with optional supporting files)
- Loaded into the agent's context via `openclaw skills install <slug>` or by placing the directory under `~/.openclaw/skills/`
- Required frontmatter: `name` (snake_case unique), `description` (one line). Optional: `homepage`, `user-invocable`, `metadata.openclaw.requires.bins`, etc. See `audit/_baseline.md` §6.
- Structure convention (from top ClawHub skills): H1 → Quick Reference → Workflow Steps → Commands → Example Session → Requirements → Configuration → Notes

### Which one to write?

| You're building | Format |
|-----------------|--------|
| "How to install system X on a fresh OpenClaw box" | `deploy-X.md` (this repo's format) |
| "A capability the agent can invoke mid-conversation" | Canonical SKILL.md (publish to ClawHub) |
| Both | Two files: the `deploy-X.md` installs + configures; the SKILL.md gets dropped into the workspace as the agent-facing skill |

Before writing a new `deploy-X.md`, check `audit/_baseline.md` §B to see if a popular ClawHub skill already solves the same problem — if yes, the `deploy-X.md` becomes a thin "install ClawHub skill Y + configure" wrapper rather than a scratch-built system.

---

## Anti-pattern addendum (added 2026-04-20 from adversarial review)

### 11. Placeholder-hand-edit in large heredocs

**Bad:**
```bash
cat > ${WORKSPACE}/docs/guide.md << 'GUIDE_EOF'
Best practices for prompting [MODEL_NAME]...
(5.5KB of content with `[MODEL_NAME]` scattered throughout)
GUIDE_EOF
# Then the skill tells the operator: "Replace [MODEL_NAME] before sending this command."
```

This forces the operator to hand-edit a large heredoc before transmission. Three failure modes: (1) operator forgets to replace, file ships with literal `[MODEL_NAME]` placeholder; (2) operator replaces one occurrence and misses others; (3) base64 encoding of the edited heredoc blows past the remote-command size ceiling.

**Good:**
```bash
# Step 1: write the body with the placeholder (quoted heredoc, no expansion needed)
cat > ${WORKSPACE}/docs/guide.md << 'GUIDE_EOF'
Best practices for prompting [MODEL_NAME]...
GUIDE_EOF

# Step 2: substitute the placeholder in place (sed runs on the remote)
sed -i "s/\[MODEL_NAME\]/${PRIMARY_MODEL}/g" ${WORKSPACE}/docs/guide.md
```

Two small commands, both mechanical. No hand-editing, no forgetting.
