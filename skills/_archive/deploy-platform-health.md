# Deploy Platform Health Council

## Purpose

Install a daily platform health analysis that reviews 9 areas of system health using AI-powered codebase analysis. Detects issues before they become problems.

**When to use:** After the core workspace is set up. Provides ongoing operational oversight.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "which agent 2>/dev/null || ls ~/.local/bin/agent 2>/dev/null"
cmd --session <id> --command "ls ${WORKSPACE}/data/cron-log.db 2>/dev/null"
cmd --session <id> --command "openclaw config get models.primary"
```
- Is the Cursor agent CLI available?
- Is the cron log database populated? (Needed for cron health checks)
- Which of the 9 analysis areas are relevant for this client?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Codebase analyzer | Cursor agent CLI at `~/.local/bin/agent` | Claude Code CLI, direct API calls | Client doesn't have Cursor agent |
| Summarizer model | Anthropic Opus | Sonnet, GPT-5.x, Gemini | Client's available AI provider |
| Analysis areas | All 9 (cron, code quality, tests, prompts, deps, storage, skills, config, data integrity) | Subset of relevant areas | Not all areas apply. Client without tests can skip test coverage. Client without CRM can skip data integrity. |
| Run schedule | Daily 4am PST | Different time, weekly, manual only | Client preference |
| CRM data integrity | Contact count drop >20% = alert | Adjust threshold, or disable | Client's CRM size and variability |
| Notification channel | Telegram Self-Improvement topic | Discord, Slack, email | Client's messaging platform |
| Deeper dives | `council-deeper-dive.js --council platform --number N` | CLI-only, different channel | Client's interaction preference |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram Self-Improvement topic)
- `deploy-security-safety.md` completed
- Cursor agent CLI installed at `~/.local/bin/agent`
- AI API access (Anthropic Opus for summarizer)
- Cron log database exists (`~/clawd/data/cron-log.db`)

## What Gets Installed

### Platform Health Script (`scripts/platform-council-v2.js`)

- Uses Cursor agent CLI for direct codebase analysis
- Opus summarizer for structured JSON output

### 9 Analysis Areas

| Area | What It Checks |
|------|---------------|
| Cron Health | Are automated jobs succeeding? Failure rates, stale jobs |
| Code Quality | Technical debt accumulating? Complexity, duplication |
| Test Coverage | Gaps in test coverage? Untested critical paths |
| Prompt Quality | Are AI prompts well-written? (per Opus prompting guide) |
| Dependencies | Outdated or vulnerable packages? |
| Storage | Databases growing too large? Disk space trends |
| Skill Integrity | Are all skills working correctly? Missing files, broken imports |
| Config Consistency | Do all config files agree with each other? Drift detection |
| Data Integrity | CRM database healthy? Contact count drops >20% = alert, corruption detection, ingestion pipeline health |

### Output Format

Numbered recommendations delivered to Telegram. Each recommendation includes:
- Area
- Severity
- Description
- Evidence
- Suggested action

Deeper-dive: `node scripts/council-deeper-dive.js --council platform --number N`

### Cron

Daily Platform Health Council -- Daily 4am PST

### Telegram Integration

- Self-Improvement topic for daily reports

## Steps

### 1. Verify Cursor Agent CLI

Confirm the Cursor agent CLI is installed and accessible.

```
cmd --session <id> --command "~/.local/bin/agent --version"
```

### 2. Install Platform Health Script

Deploy the platform-council-v2.js script to the scripts directory.

```
cmd --session <id> --command "ls ~/clawd/scripts/platform-council-v2.js"
```

If not present, install it:

```
cmd --session <id> --command "openclaw skill install platform-health --handler ~/clawd/scripts/platform-council-v2.js"
```

### 3. Run Manual Test (Dry Run)

Execute a full platform health analysis without delivering to Telegram.

```
cmd --session <id> --command "cd ~/clawd && node scripts/platform-council-v2.js --dry-run --json"
```

Review the output:
- All 9 analysis areas should produce results
- Each recommendation should have area, severity, description, evidence, suggested action
- Recommendations should be numbered sequentially

### 4. Set Up Daily Cron at 4am PST

```
cmd --session <id> --command "openclaw cron add --name 'Daily Platform Health Council' --schedule '0 4 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd && node scripts/platform-council-v2.js'"
```

### 5. Configure Telegram Delivery

Route reports to the Self-Improvement topic.

```
cmd --session <id> --command "openclaw config set platform-health.telegram-topic self-improvement"
```

## Verification

Run these commands to confirm the platform health council is operational:

```
cmd --session <id> --command "cd ~/clawd && node scripts/platform-council-v2.js --dry-run --json"
```

Expected output:
- Structured JSON with recommendations from all 9 analysis areas
- Each recommendation should be actionable with clear evidence
- Cron health area should reflect actual cron job status from `~/clawd/data/cron-log.db`

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Cursor agent not found | CLI not at expected path | Verify `~/.local/bin/agent` exists. If installed elsewhere, create a symlink: `ln -s /actual/path ~/.local/bin/agent`. |
| Missing cron-log data | No cron jobs have run yet | Run some cron jobs first to populate `~/clawd/data/cron-log.db`. The cron health analyzer needs historical data to calculate failure rates. |
| Contact count alert (false positive) | CRM was just deployed, no baseline exists | The data integrity check compares current count to 7-day average. If the CRM was just deployed, the baseline hasn't been established. Ignore for the first week. |
| Slow execution | Full codebase analysis is expensive | Normal execution is 10-20 minutes depending on codebase size. The Cursor agent CLI analyzes actual code, not just metadata. |
| Opus API errors | ANTHROPIC_API_KEY missing or rate limited | Verify the key in `~/.openclaw/.env`. The summarizer specifically needs Opus-tier access for structured analysis. |
| Dependencies area empty | No package.json or requirements.txt found | The dependency checker scans for known package manifest files. If the project uses an unusual layout, configure the scan paths manually. |
| Prompt quality area skipped | No prompt files found | The prompt analyzer looks for files matching `*prompt*`, `*system*`, `SOUL.md`, `IDENTITY.md`. If prompts are stored elsewhere, configure the scan paths. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Enhanced by:** `deploy-personal-crm.md` (CRM data integrity checks), cron jobs running (cron health data)
