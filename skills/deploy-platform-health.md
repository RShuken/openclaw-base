# Deploy Platform Health Council

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-19 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale; `openclaw cron` + `openclaw config` subcommands exist in 2026.4.15 per `audit/_baseline.md` §3 and §4C.7
- **Scheduling**: `openclaw cron add --name daily-platform-health --cron "0 5 * * *"` (native, no launchd/crontab fallback needed)
- **Config**: `openclaw config set <key> <value>` for scalars; edit `~/.openclaw/openclaw.json` directly with Python for arrays (`config set` can't set arrays — documented limitation)
- **Overlap with bundled `doctor`**: `openclaw doctor` covers basic health checks out of the box. This skill's 9-area deep codebase analysis is the delta. Don't duplicate what `doctor` already does — extend it.
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`HEALTH_ANALYSIS_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Route the 9-area analysis to a cheaper model here to bring per-run cost down.

## Purpose

Install a daily platform health analysis that reviews up to 9 areas of system health using AI-powered codebase analysis. Each area produces numbered, actionable recommendations with severity, evidence, and suggested fixes. Detects issues before they become problems, delivering results to the client's notification channel.

**When to use:** After the core workspace and messaging are set up. Provides ongoing operational oversight for deployed OpenClaw systems.

**What this skill does:**
1. Verifies a codebase analysis tool is available (Cursor agent CLI, Claude Code, or direct API)
2. Deploys the platform health analysis script to the workspace
3. Runs a dry-run test to confirm all analysis areas produce output
4. Schedules a daily cron job for automated health analysis
5. Configures notification delivery to the client's preferred channel

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${AGENT_CLI}` | Pre-flight check: path to codebase analysis CLI | `~/.local/bin/agent` |
| `${SUMMARIZER_MODEL}` | Client profile: available AI models | `claude-sonnet-4-6` |
| `${HEALTH_TOPIC_ID}` | Created during messaging setup, or client profile: messaging config | `1207` |
| `${NOTIFICATION_CHANNEL}` | Client profile: preferred notification service | `telegram` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${CRON_HOUR}` | Client preference or default | `4` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if a codebase analysis CLI is available (Cursor agent, Claude Code, or other):

**Remote:**
```
which agent 2>/dev/null || ls ~/.local/bin/agent 2>/dev/null || which claude 2>/dev/null || echo 'NO_ANALYZER'
```

Expected: Path to a codebase analysis CLI.

If this fails: The platform health script needs a tool that can analyze code. If neither `agent` nor `claude` is available, the client needs one installed, or the script must be adapted to use direct API calls instead.

Check if the cron log database exists (needed for cron health analysis area):

**Remote:**
```
ls ${WORKSPACE}/data/cron-log.db 2>/dev/null || echo 'NO_CRON_DB'
```

Expected: File path printed. If `NO_CRON_DB`, the cron health analysis area will have no data -- this is acceptable for initial deployment, it will populate as cron jobs run.

Check the client's primary AI model configuration:

**Remote:**
```
openclaw config get models.primary 2>/dev/null || echo 'NO_MODEL_CONFIG'
```

Expected: Model identifier. This determines which summarizer model the health script uses.

Check if the platform health script already exists:

**Remote:**
```
ls ${WORKSPACE}/scripts/platform-council-v2.js 2>/dev/null || echo 'NOT_DEPLOYED'
```

Expected: If file exists, check if it needs updating. If `NOT_DEPLOYED`, proceed with full installation.

**Decision points from pre-flight:**
- Which codebase analyzer is available? This determines the `${AGENT_CLI}` variable.
- Which of the 9 analysis areas are relevant? (Client without tests can skip test coverage; client without CRM can skip data integrity.)
- Is there an existing platform health script that needs updating vs. fresh install?
- Does the cron log database exist? If not, the cron health area won't produce meaningful results initially.

## Adaptation Points

The defaults below work for most installs. Cross-reference against the client profile for overrides.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Codebase analyzer | Cursor agent CLI at `~/.local/bin/agent` | Client has Claude Code, a different CLI, or only API access |
| Summarizer model | Anthropic Opus | Client has a different AI provider or model tier preference |
| Analysis areas | All 9 (cron, code quality, tests, prompts, deps, storage, skills, config, data integrity) | Not all areas apply -- client without tests skips test coverage, client without CRM skips data integrity |
| Run schedule | Daily at 4am in client's timezone | Client prefers a different time, weekly cadence, or manual-only |
| CRM data integrity threshold | Contact count drop >20% triggers alert | Client's CRM size is small (volatile) or large (stable) -- adjust threshold accordingly |
| Notification channel | Telegram Self-Improvement topic | Client uses Discord, Slack, email, or a different Telegram topic |
| Deeper-dive interaction | `node ${WORKSPACE}/scripts/council-deeper-dive.js --council platform --number N` | Client prefers CLI-only or a different interaction model |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed (notification topic exists)
- `deploy-security-safety.md` completed
- A codebase analysis tool available (Cursor agent CLI, Claude Code CLI, or direct API access)
- AI API access for the summarizer (Anthropic key with Opus-tier access, or equivalent)

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| Platform health script | `${WORKSPACE}/scripts/platform-council-v2.js` | Runs 9 analysis areas using codebase analyzer + AI summarizer |
| Deeper-dive script | `${WORKSPACE}/scripts/council-deeper-dive.js` | On-demand deep analysis of a specific recommendation |
| Cron job | `Daily Platform Health Council` | Runs the health script on schedule |
| Notification config | OpenClaw config | Routes health reports to the correct notification topic |

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

Numbered recommendations delivered to the notification channel. Each recommendation includes:
- Area
- Severity (critical / warning / info)
- Description
- Evidence
- Suggested action

## Steps

### Phase 1: Verify Analyzer and Environment

#### 1.1 Confirm Codebase Analyzer `[AUTO]`

Verify the codebase analysis CLI is installed, accessible, and responds to a version check. This is the tool the platform health script delegates code analysis to.

**Remote:**
```
${AGENT_CLI} --version
```

Expected: Version string output (e.g., `0.1.x` for Cursor agent, or version for Claude Code).

If this fails: The CLI may be installed at a non-standard path. Search for it:

**Remote:**
```
find /usr/local/bin ~/.local/bin /opt -name "agent" -o -name "claude" 2>/dev/null
```

If no codebase analyzer is found, the client needs one installed before this skill can be deployed. Alternatively, adapt the platform health script to use direct API calls (Level 3 adaptation -- discuss with operator).

#### 1.2 Verify Workspace Structure `[AUTO]`

Ensure the scripts directory exists in the workspace.

**Remote:**
```
mkdir -p ${WORKSPACE}/scripts
```

Expected: Directory exists (or is created). `mkdir -p` is idempotent.

#### 1.3 Check AI API Access `[AUTO]`

Verify the AI API key is configured and the summarizer model is accessible. The platform health script uses an AI model to produce structured JSON from raw analysis output.

**Remote:**
```
test -f ${WORKSPACE}/.env && grep -q "ANTHROPIC_API_KEY\|OPENAI_API_KEY\|GOOGLE_API_KEY" ${WORKSPACE}/.env && echo 'API_KEY_FOUND' || echo 'NO_API_KEY'
```

Expected: `API_KEY_FOUND`. The specific key depends on which summarizer model the client uses.

If this fails: The client needs an API key configured. Check with the operator which AI provider is available and add the key to `${WORKSPACE}/.env`.

### Phase 2: Deploy Health Script

#### 2.1 Install Platform Health Script `[GUIDED]`

Deploy the platform-council-v2.js script to the workspace scripts directory. This script orchestrates all 9 analysis areas and produces the structured report.

First, check if the script already exists:

**Remote:**
```
ls -la ${WORKSPACE}/scripts/platform-council-v2.js 2>/dev/null
```

If not present, the script needs to be created. The script should:
- Accept `--dry-run` flag (skip notification delivery, print to stdout)
- Accept `--json` flag (output structured JSON)
- Accept `--areas` flag (comma-separated list to run a subset of analysis areas)
- Use `${AGENT_CLI}` for codebase analysis tasks
- Use the configured summarizer model for structured output generation
- Read cron data from `${WORKSPACE}/data/cron-log.db` for the cron health area
- Read CRM data from `${WORKSPACE}/data/personal-crm.db` for the data integrity area (if it exists)
- Scan for `package.json`, `requirements.txt`, and similar manifests for the dependencies area
- Scan for files matching `*prompt*`, `*system*`, `SOUL.md`, `IDENTITY.md` for the prompt quality area

**Remote:**
```
ls ${WORKSPACE}/scripts/platform-council-v2.js
```

Expected: File exists at `${WORKSPACE}/scripts/platform-council-v2.js`.

If already exists: Compare the existing script against expected behavior. If it matches, skip. If it is outdated, back up as `platform-council-v2.js.bak` and deploy the updated version.

If this fails: The script file was not created successfully. Check write permissions on `${WORKSPACE}/scripts/`. Also verify the script content is valid JavaScript by running `node --check ${WORKSPACE}/scripts/platform-council-v2.js`.

#### 2.2 Install Deeper-Dive Script `[AUTO]`

Deploy the council-deeper-dive.js script that allows on-demand deep analysis of a specific recommendation from the daily report.

Check if it already exists:

**Remote:**
```
ls ${WORKSPACE}/scripts/council-deeper-dive.js 2>/dev/null || echo 'NOT_PRESENT'
```

If not present, deploy it. The deeper-dive script should:
- Accept `--council platform` to target platform health recommendations
- Accept `--number N` to select a specific recommendation by number
- Perform an extended analysis of the selected recommendation
- Output the deeper analysis to the notification channel or stdout

Expected: File exists at `${WORKSPACE}/scripts/council-deeper-dive.js`.

If already exists: Same idempotency pattern -- compare, skip if current, backup and update if outdated.

### Phase 3: Test and Schedule

#### 3.1 Run Dry-Run Test `[GUIDED]`

Execute a full platform health analysis in dry-run mode to verify all analysis areas produce output without delivering to the notification channel.

**Remote:**
```
cd ${WORKSPACE} && node scripts/platform-council-v2.js --dry-run --json
```

Expected: Structured JSON output with recommendations from the relevant analysis areas. Each recommendation should contain:
- `area` (one of the 9 analysis areas)
- `severity` (critical / warning / info)
- `description` (what was found)
- `evidence` (supporting data)
- `suggestedAction` (what to do about it)

Recommendations should be numbered sequentially.

If this fails:
- **"Cannot find module" errors:** Check that all dependencies are installed. Run `cd ${WORKSPACE} && npm install` if there is a package.json, or check that the script's imports are available.
- **Analyzer timeout:** Full codebase analysis can take 10-20 minutes depending on codebase size. This is normal. If it times out, try running with a subset: `node scripts/platform-council-v2.js --dry-run --json --areas cron,storage,deps`
- **Empty results for some areas:** This is acceptable. Areas that have no data (e.g., cron health before any cron jobs have run, test coverage on a project without tests) will produce empty results.
- **API errors from summarizer:** Verify the API key in `${WORKSPACE}/.env` and check that the configured model is accessible.

#### 3.2 Schedule Daily Cron Job `[GUIDED]`

Set up the daily cron job to run the platform health analysis automatically.

First, check if a platform health cron job already exists:

**Remote:**
```
openclaw cron list 2>/dev/null | grep -i "platform health" || echo 'NO_EXISTING_CRON'
```

If no existing cron, add one:

**Remote:**
```
openclaw cron add --name 'Daily Platform Health Council' --schedule '0 ${CRON_HOUR} * * *' --tz '${TIMEZONE}' \
  --env HEALTH_ANALYSIS_MODEL=openai-codex/gpt-5.4-nano \
  --command 'cd ${WORKSPACE} && node scripts/platform-council-v2.js'
```

`HEALTH_ANALYSIS_MODEL` routes the 9-area analysis LLM calls. Omit to defer to the agent default. In `scripts/platform-council-v2.js`:

```javascript
const MODEL = process.env.HEALTH_ANALYSIS_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

Expected: Cron job created confirmation. The job should appear in `openclaw cron list`.

If this fails: Check that `openclaw cron` is available. If the client uses system crontab instead, add the entry directly:

**Remote:**
```
(crontab -l 2>/dev/null; echo "0 ${CRON_HOUR} * * * cd ${WORKSPACE} && node scripts/platform-council-v2.js") | crontab -
```

If already exists: Verify the schedule and command match what we want. If they differ, update. If they match, skip.

#### 3.3 Configure Notification Delivery `[GUIDED]`

Route platform health reports to the client's notification channel and topic.

**Remote:**
```
openclaw config set platform-health.telegram-topic ${HEALTH_TOPIC_ID}
```

Expected: Configuration updated confirmation.

If this fails: Check that `openclaw config` is available and that the topic ID is valid. Verify the messaging setup was completed (`deploy-messaging-setup.md`).

If already configured: Check the current value. If it matches, skip. If different, update to the new value.

## Verification

Run these checks to confirm the platform health council is fully operational.

**Verify the script exists and is valid:**

**Remote:**
```
node --check ${WORKSPACE}/scripts/platform-council-v2.js && echo 'VALID'
```

Expected: `VALID` -- the script parses without syntax errors.

**Verify the cron job is registered:**

**Remote:**
```
openclaw cron list | grep -i "platform health"
```

Expected: Shows the daily platform health job with the correct schedule.

**Verify notification config:**

**Remote:**
```
openclaw config get platform-health.telegram-topic
```

Expected: Returns the configured topic ID.

**Full end-to-end dry run:**

**Remote:**
```
cd ${WORKSPACE} && node scripts/platform-council-v2.js --dry-run --json
```

Expected: Structured JSON with recommendations from all relevant analysis areas. Cron health area should reflect actual cron job status from `${WORKSPACE}/data/cron-log.db` if it exists.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Codebase analyzer not found | CLI not at expected path | Search for it: `find /usr/local/bin ~/.local/bin /opt -name "agent" -o -name "claude" 2>/dev/null`. If installed elsewhere, update `${AGENT_CLI}` or create a symlink. |
| Missing cron-log data | No cron jobs have run yet | The cron health area needs historical data. Run some cron jobs first to populate `${WORKSPACE}/data/cron-log.db`. Ignore empty cron health results for the first few days. |
| Contact count alert (false positive) | CRM was just deployed, no baseline exists | The data integrity check compares current count to 7-day average. If CRM was just deployed, ignore for the first week while the baseline is established. |
| Slow execution (>20 min) | Full codebase analysis is expensive | Normal execution is 10-20 minutes. If too slow, run a subset of areas with `--areas cron,storage,deps` to skip expensive code analysis. |
| AI API errors | API key missing or rate limited | Verify the key in `${WORKSPACE}/.env`. The summarizer needs access to the configured model (e.g., Opus-tier for Anthropic). |
| Dependencies area empty | No package manifest files found | The dependency checker scans for `package.json`, `requirements.txt`, `Cargo.toml`, etc. If the project uses an unusual layout, configure scan paths in the script. |
| Prompt quality area skipped | No prompt files found | The prompt analyzer looks for files matching `*prompt*`, `*system*`, `SOUL.md`, `IDENTITY.md`. If prompts are stored elsewhere, configure the scan paths in the script. |
| Script syntax error on deploy | Script content was corrupted during transfer | Run `node --check ${WORKSPACE}/scripts/platform-council-v2.js` to see the parse error. Redeploy the script. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Confirm codebase analyzer | Execute silently | Execute silently | Confirm before running |
| 1.2 Verify workspace structure | Execute silently | Execute silently | Confirm before creating directory |
| 1.3 Check AI API access | Execute silently | Execute silently | Confirm before checking |
| 2.1 Install platform health script | Execute silently | Confirm before deploying script | Confirm, show script content |
| 2.2 Install deeper-dive script | Execute silently | Execute silently | Confirm before deploying |
| 3.1 Run dry-run test | Execute, report results | Execute, review results together | Confirm before running, review results |
| 3.2 Schedule daily cron | Execute silently | Confirm schedule and timezone | Confirm the exact cron expression |
| 3.3 Configure notifications | Execute silently | Confirm topic/channel | Confirm the exact config value |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Enhanced by:** `deploy-personal-crm.md` (enables CRM data integrity checks), any deployed cron jobs (provides data for cron health analysis)
- **Required by:** Nothing directly -- this is an observability/monitoring system
