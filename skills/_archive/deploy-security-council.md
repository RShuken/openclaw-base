# Deploy Security Council

## Purpose

Install an automated nightly security review that analyzes the entire codebase from four security perspectives using AI. Produces structured findings delivered to Telegram with deeper-dive capability.

**When to use:** After the core workspace is set up. Provides ongoing security oversight.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/scripts/security-review.sh 2>/dev/null"
cmd --session <id> --command "which agent 2>/dev/null || ls ~/.local/bin/agent 2>/dev/null"
cmd --session <id> --command "openclaw config get models.primary"
```
- Is the security-review.sh baseline script installed? (Part of deploy-security-safety.md)
- Is the Cursor agent CLI available? (Used for codebase analysis)
- What AI model is available for the summarizer?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Codebase analyzer | Cursor agent CLI at `~/.local/bin/agent` | Claude Code CLI, direct API calls, or skip deep analysis | Client doesn't have Cursor agent installed |
| Summarizer model | Anthropic Opus | Sonnet, GPT-5.x, Gemini | Client's available AI provider |
| Run schedule | Nightly 3:30am PST | Different time, weekly, manual only | Client preference or cost concerns |
| Security perspectives | All 4 (offensive, defensive, data privacy, operational realism) | Subset | Smaller codebase may not need all perspectives |
| Critical alerts | Immediate Telegram DM | Discord DM, Slack DM, email | Client's preferred alert channel |
| Report delivery | Telegram Self-Improvement topic | Different topic/channel | Client's messaging setup |
| Baseline checks | Full security-review.sh suite | Subset of checks | Some checks may not apply (e.g., gateway check if no gateway) |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram Self-Improvement topic for reports)
- `deploy-security-safety.md` completed (security-review.sh baseline)
- Cursor agent CLI installed at `~/.local/bin/agent` (for codebase analysis)
- AI API access (Anthropic Opus for summarizer)

## What Gets Installed

### Security Council Script (`scripts/security-council-v2.js`)

- Runs `security-review.sh` for baseline checks
- Uses Cursor agent CLI for direct codebase analysis (reads actual code, not static rules)
- Opus summarizer for structured JSON output

### Four Analysis Perspectives

| Perspective | What It Checks |
|-------------|---------------|
| Offensive | What could an attacker exploit? Entry points, injection vectors, privilege escalation |
| Defensive | Are protections adequate? Auth, encryption, input validation, rate limiting |
| Data Privacy | Is sensitive data handled correctly? PII exposure, logging, storage, transmission |
| Operational Realism | Are security measures practical or just theater? False positive rates, usability impact |

### Baseline Checks (from security-review.sh)

- File permissions (.env, .db files, openclaw.json, system prompts)
- Gateway binds to loopback only (not exposed to internet)
- Auth enabled on gateway
- No secrets in git-tracked files
- Security modules wired in (content-sanitizer, secret-redaction)
- Backup encryption
- Prompt injection patterns in config
- .gitignore rules intact
- Error log review for auth failures

### Output Format

Structured JSON report with numbered findings. Each finding includes:
- Severity (critical/high/medium/low)
- Category
- Description
- Evidence
- Recommendation

Critical findings alert immediately to Telegram DM. All findings support deeper-dive requests.

### Cron

Nightly Security Review -- Daily 3:30am PST

### Telegram Integration

- Self-Improvement topic for nightly reports
- DM for critical severity alerts (immediate)
- Deeper-dive: `node scripts/council-deeper-dive.js --council security --number N`

## Steps

### 1. Verify Security Review Script Exists and Runs

The baseline security-review.sh should already be deployed from `deploy-security-safety.md`.

```
cmd --session <id> --command "./scripts/security-review.sh"
```

Confirm it runs without errors and produces a report.

### 2. Verify Cursor Agent CLI

The Cursor agent CLI is used for deep codebase analysis. Confirm it is installed and accessible.

```
cmd --session <id> --command "~/.local/bin/agent --version"
```

### 3. Install Security Council Script

Deploy the security-council-v2.js script to the scripts directory.

```
cmd --session <id> --command "ls ~/clawd/scripts/security-council-v2.js"
```

If not present, install it from the skill package:

```
cmd --session <id> --command "openclaw skill install security-council --handler ~/clawd/scripts/security-council-v2.js"
```

### 4. Run Manual Test (Dry Run)

Execute a full security review without delivering to Telegram. Inspect the JSON output for correct structure.

```
cmd --session <id> --command "cd ~/clawd && node scripts/security-council-v2.js --dry-run --json"
```

Review the output:
- All four perspectives should produce findings
- Each finding should have severity, category, description, evidence, recommendation
- Findings should be numbered sequentially

### 5. Set Up Nightly Cron at 3:30am PST

```
cmd --session <id> --command "openclaw cron add --name 'Nightly Security Review' --schedule '30 3 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd && node scripts/security-council-v2.js'"
```

### 6. Configure Telegram Delivery

Route reports to the Self-Improvement topic and critical alerts to DM.

```
cmd --session <id> --command "openclaw config set security-council.telegram-topic self-improvement"
cmd --session <id> --command "openclaw config set security-council.critical-alert-mode dm"
```

## Verification

Run these commands to confirm the security council is operational:

```
cmd --session <id> --command "./scripts/security-review.sh"
cmd --session <id> --command "cd ~/clawd && node scripts/security-council-v2.js --dry-run --json"
```

Expected output:
- security-review.sh should complete with pass/fail for each baseline check
- security-council-v2.js should produce structured JSON with findings from all four perspectives
- Each finding should be actionable with clear evidence and recommendations

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Cursor agent not found | CLI not at expected path | Verify `~/.local/bin/agent` exists. If installed elsewhere, create a symlink: `ln -s /actual/path ~/.local/bin/agent`. |
| Security review script fails | Missing lib/security-review-checks.sh | Verify `deploy-security-safety.md` was completed. The baseline script depends on the checks library. |
| Empty findings | No issues detected | May actually mean the system is secure. Verify by manually checking one area (e.g., file permissions) to confirm the analyzers are running. |
| Slow execution | Large codebase analysis | Codebase analysis via agent CLI takes time, especially for large repos. Normal execution is 5-15 minutes. Set a generous cron timeout. |
| Opus API errors | ANTHROPIC_API_KEY missing or rate limited | Verify the key in `~/.openclaw/.env`. The summarizer specifically needs Opus-tier access. |
| Critical alert not delivered | DM mode not configured | Verify `security-council.critical-alert-mode` is set to `dm` and the bot has DM access to the operator. |
| Duplicate findings across runs | No deduplication | The council deduplicates within a single run but not across runs. Known findings that persist are intentional (they haven't been fixed). |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Enhanced by:** Having more code to analyze (more code = more comprehensive review)
