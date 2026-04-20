# Skill Review Protocol

How to review a skill before it's considered ready. Every new or modified skill passes this review.

> **Rewritten 2026-04-20.** Prior version prescribed "launch 3 agents in parallel" for Format / Accuracy / Operability review. That's the exact anti-pattern that burned us in earlier audit work — subagents routinely claim "I reviewed all 12 items" when they actually skipped several. Reviews are now done **directly by Claude**, one item at a time, with visible tool-call evidence. Subagents are allowed only as a **harness** where Claude verifies their findings against the actual files before trusting them.

## When to review

- **New skill created** — full review before first use
- **Skill modified** — full review of the modified skill
- **Periodic audit** — all skills reviewed quarterly or when OpenClaw ships a breaking minor release

## Method

Claude opens the skill file with the Read tool (the Read call is the audit-trail — visible in the transcript). Claude grades against the three lenses below, writing a finding per issue. For any non-trivial issue, quote the specific lines.

**Do NOT delegate this to a subagent without a harness.** If speed requires fan-out, the harness pattern is:

1. Subagent produces findings with exact line-number citations
2. Claude re-opens the cited lines and verifies the content matches the claim
3. Verified findings roll into the report; unverified/mismatched claims are dropped

See `~/.claude/projects/-Users-shuken-AI-proletariat/memory/feedback_no_subagent_audits.md` for the durable version of this rule.

---

## Lens 1 — Format & Structure

Does the skill follow the conventions in `_skill-authoring-guide.md`?

**Compat block present:**
- `## Compatibility` section at the top with: OpenClaw Version, Status, Scheduling strategy, Config strategy, Model note
- Version header is current (2026.4.15+ as of this writing; check `audit/_baseline.md` §1)
- **Status is not lying** — if it says WORKING, the skill actually works on the stated version. Common trap: old "BLOCKED" claims against commands that shipped in a later release.

**Body structure:**
- `## Purpose` — one paragraph + "When to use"
- `## Variables` — table with Variable / Source / Example. Every `${VAR}` used below is listed here.
- `## Before You Start` — pre-flight checks
- `## Adaptation Points` — what can be customized
- `## Prerequisites`
- `## What Gets Installed`
- `## Steps` — numbered phases with step-by-step instructions
- `## Verification` — concrete checks
- `## Troubleshooting` — Problem / Cause / Fix table
- `## Autonomy Mode Behavior` — Step / Auto / Guided / Manual table
- `## Dependencies` — Depends on / Required by / Enhanced by

**Cross-platform awareness:**
- Platform-specific sequences (daemon registration, service configs) include exact syntax per platform
- Everything else uses intent + bash reference per the 3-level model

**Variable hygiene:**
- No hardcoded topic IDs, project IDs, chat IDs, emails, or client names
- Every placeholder defined in the Variables table or `_deploy-common.md`
- No fictional CLI (`cmd --session <id>` or similar)

**Consistency:**
- `${WORKSPACE}` everywhere, not `~/clawd/`
- Same variable name for the same concept throughout

---

## Lens 2 — Technical Accuracy

Are the commands correct on OpenClaw 2026.4.15?

**OpenClaw CLI usage:**
- Every `openclaw <subcommand>` referenced is in the live inventory (`audit/_baseline.md` §4C.7)
- Common gotchas: `openclaw status` does NOT exist (use `openclaw health`); `openclaw cron rm` not `remove`; `openclaw skills` not `openclaw skill`; `openclaw channels` replaces old `openclaw telegram`/`slack` subcommands
- Scheduling uses `openclaw cron add` where possible — falling back to launchd/crontab/schtasks only with explicit justification

**Model references:**
- No hardcoded retiring snapshot IDs (baseline §X)
- Skill defers to `agents.defaults.model` when it routes through an LLM, rather than hardcoding `anthropic` or `openai-codex`
- When a specific model IS required (e.g., Haiku for cheap classification), pin it by stable alias (`claude-haiku-4-5`, `gpt-5.3-codex`) with a comment explaining why

**Shell quoting + heredocs:**
- Heredocs: `<< 'EOF'` suppresses expansion; `<< EOF` allows it. Picking the wrong one is a classic bug.
- No `echo '\n'` — use `printf '\n'` or `echo ""`
- No `echo -e` — use `printf` for escape sequences
- No `for file in $(cmd)` — use `while IFS= read -r file` for filenames with spaces
- `wc -c` on macOS has leading whitespace — pipe through `tr -d ' '` or use `stat`

**Quoted-heredoc placeholder anti-pattern:**
Skills that write a file with placeholder text inside a **single-quoted** heredoc and then ask the operator to hand-edit the placeholder before transmission are bug-prone (the placeholder gets written literally; the substitution often gets forgotten). Instead, write the body first and `sed -i` the placeholders in a follow-up step.

**Credentials:**
- API keys live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, NOT in `~/.openclaw/.env` (which is only read by custom scripts)
- Never migrate OpenClaw secrets to macOS Keychain under LaunchAgent — see `_deploy-common.md` "Secret storage" section for the Keychain-incident lesson

**Command correctness:**
- macOS: `launchctl bootstrap gui/$(id -u)` not deprecated `launchctl load`
- Linux: usernames resolved to literal values before being written to systemd unit files
- Windows: `schtasks /IT` for interactive session access

---

## Lens 3 — AI Operability

Can the agent actually execute this skill?

**Variable resolution:**
- Every `${VARIABLE}` has a documented source
- Resolution order is explicit where it matters
- No "check the dashboard" instructions — the agent should be able to find everything programmatically

**Intent-based guidance:**
- Steps describe WHAT to accomplish and WHY, not just commands to paste
- Level 1 (exact syntax) only for critical sequences
- Level 2–3 (intent + reference) for standard operations

**Error handling:**
- Each step has `Expected:` outcome AND `If this fails:` guidance
- Specific remediation, not generic "try again"
- Clear rollback path if a phase fails partway through

**Autonomy markers:**
- `[AUTO]` / `[GUIDED]` / `[HUMAN_INPUT]` on every step
- `[HUMAN_INPUT]` only for API keys, passwords, verification codes — things the agent genuinely can't resolve

**Idempotency:**
- Re-running the skill doesn't break anything
- Pre-flight checks detect existing installations
- "If already exists:" cases have clear handling (skip / backup+overwrite / error)

**Cost awareness:**
- Skills that run on a schedule or in heartbeat context note their per-run cost order-of-magnitude
- Skills that fire multiple LLM calls per run flag the cost explicitly
- Heartbeat-triggered LLM calls ≤ 1 per alert (cheap checks first — see baseline §L)

---

## Severity & output

When reviewing, grade each finding:

| Severity | Meaning |
|----------|---------|
| CRITICAL | Command will fail, data will be lost, security vulnerability, or hardcoded retiring model ID |
| HIGH | Likely to cause confusion or require manual intervention |
| MEDIUM | Inconsistency or missing information that degrades quality |
| LOW | Style nit or minor improvement |

Write findings into the skill's commit message, the PR description, or `audit/skill-audit-<date>.md` if reviewing at scale.

## Report template

```
## Skill Review: <skill-name>

### Summary
- Format: X issues (Y CRITICAL, Z HIGH, W MEDIUM)
- Accuracy: X issues
- Operability: X issues

### CRITICAL (must fix before merge)
1. [ACCURACY] L<line>: <issue>. Fix: <specific fix>.

### HIGH (should fix)
1. [FORMAT] L<line>: <issue>. Fix: ...

### MEDIUM / LOW
...

### Verdict
KEEP / REWRITE (light|medium|heavy) / REPLACE-WITH-CLAWHUB / DELETE
```

## What NOT to do in a review

- **Don't delegate the per-skill verdict to a subagent without verification.** Subagents claim "done" without reading; the harness pattern (Claude verifies citations) is the only safe way.
- **Don't score against a baseline you haven't written down.** Reviews need a standards doc to grade against (`audit/_baseline.md` is the one for this repo). "Claude thinks this looks stale" is not a valid finding.
- **Don't approve a skill that hardcodes a deprecated model ID.** That's a CRITICAL regardless of how polished the rest is — it's a scheduled breakage.
- **Don't approve a skill that fires N cron jobs when a single heartbeat would work.** §O community anti-pattern.
- **Don't approve a skill that leaks client identifiers** (emails, chat IDs, group IDs, company names) into what's supposed to be a base library.
