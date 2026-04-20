# Deploy Model Usage & Cost Tracking

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — every previously-claimed "Blocked Command" was wrong. On 2026.4.15:
  - `openclaw config get/set` ✅ valid per §4C.7
  - `openclaw skills install` ✅ valid per §4C.4 (it's `skills` plural, not `skill`)
  - `openclaw doctor` ✅ valid per §4C.7
- **Scheduling**: `openclaw cron add --name model-tracking-rollup --cron "0 * * * *"` (hourly log rollup)
- **Unique value**: no ClawHub equivalent for model-cost dashboards per §B — worth keeping
- **Model**: agent default (Codex-first); do not hardcode

## Purpose

Install a usage tracker that logs every AI API call across all providers. Tracks model, tokens, task type, and estimated cost. Generates reports to monitor spending. The system tracks its own expenses so the user always knows how much it costs to run — every AI call made by the agent (CRM queries, email classification, meeting processing, research) is logged.

**When to use:** Any time AI APIs are being used. Deploy early to track costs from the start.

**What this skill does:**
1. Creates the log directory and initializes the JSONL log file
2. Installs the model-usage-tracker skill (log-usage.js + generate-report.js)
3. Logs a test entry and generates a test report
4. Enables model tracking in the agent middleware configuration

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${OPENCLAW_CONFIG}` | Client profile: config path | `~/.openclaw/` |
| `${LOG_PATH}` | Adaptation point; default `${OPENCLAW_CONFIG}/logs/model-usage.jsonl` | `~/.openclaw/logs/model-usage.jsonl` |
| `${REPORT_CHANNEL}` | Client profile: preferred notification channel (if automated reports desired) | `telegram` or `slack` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if a usage log already exists (upgrade vs fresh install):

**Remote:**
```
ls ${OPENCLAW_CONFIG}/logs/model-usage.jsonl 2>/dev/null || echo 'NO_LOG'
```

Expected: Either the file path (existing log) or `NO_LOG` (fresh install).

Check which AI providers the client has configured:

**Remote:**
```
openclaw config get models
```

Expected: List of configured providers (e.g., Anthropic, OpenAI, Google, xAI).

If this fails: OpenClaw may not be installed yet. Deploy `openclaw/install.md` first.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Log format | JSONL at `${OPENCLAW_CONFIG}/logs/model-usage.jsonl` | Client wants a queryable database (SQLite) instead of flat file |
| Providers tracked | All (Anthropic, OpenAI, Google, xAI) | Client only uses some providers |
| Cost estimation | Built-in token pricing | Client has negotiated rates or subscription pricing |
| Report delivery | CLI on demand | Client wants automated daily/weekly reports to Telegram/Slack |
| Cost alerts | None (manual review) | Client wants spend guardrails (alert when daily/weekly spend exceeds threshold) |

## Prerequisites

- OpenClaw agent installed and running (`openclaw/install.md`)
- At least one AI provider configured

## What Gets Installed

### Model Usage Tracker Skill (`skills/model-usage-tracker/`)

| Script | Purpose |
|--------|---------|
| `log-usage.js` | Log a single API call: model, tokens_in, tokens_out, task_type, description |
| `generate-report.js` | Generate cost reports with filters: `--days N`, `--model X`, `--task-type Y` |

### Storage

JSONL log at `${OPENCLAW_CONFIG}/logs/model-usage.jsonl`.

Each log entry:

```json
{
  "timestamp": "2026-02-23T10:30:00Z",
  "model": "anthropic/claude-opus-4-6",
  "tokens_in": 5000,
  "tokens_out": 1200,
  "task_type": "conversation",
  "description": "CRM query",
  "estimated_cost": 0.033
}
```

### Fields

| Field | Purpose |
|-------|---------|
| `timestamp` | ISO 8601 timestamp of the API call |
| `model` | Provider/model identifier (e.g., `anthropic/claude-opus-4-6`, `google/gemini-2.0-flash`) |
| `tokens_in` | Input tokens consumed |
| `tokens_out` | Output tokens generated |
| `task_type` | Category: conversation, embedding, classification, research, summarization, code |
| `description` | Brief description of what the call was for |
| `estimated_cost` | USD cost estimate based on current provider pricing |

### Report Flags

| Flag | Purpose |
|------|---------|
| `--days N` | Show last N days (default: 7) |
| `--model X` | Filter by model name or provider |
| `--task-type Y` | Filter by task type |
| `--format table\|json\|csv` | Output format (default: table) |
| `--group-by model\|task-type\|day` | Group results (default: model) |

Example report output:

```
Model Usage Report (Last 7 Days)
================================
Model                          Calls   Tokens In   Tokens Out   Est. Cost
anthropic/claude-opus-4-6       142     850,000     210,000      $14.20
google/gemini-2.0-flash         89      420,000     180,000      $1.85
google/gemini-embedding-001     234     1,200,000   0            $0.12
--------------------------------
Total                           465     2,470,000   390,000      $16.17
```

## Steps

### Phase 1: Set Up Log Storage

#### 1.1 Create Log Directory `[AUTO]`

Create the directory where usage logs will be stored.

**Remote:**
```
mkdir -p ${OPENCLAW_CONFIG}/logs
```

Expected: Directory exists at `${OPENCLAW_CONFIG}/logs/`. No output on success.

If this fails: Check that `${OPENCLAW_CONFIG}` exists. If not, OpenClaw may not be installed. Run `openclaw/install.md` first.

If already exists: `mkdir -p` is idempotent. Safe to re-run.

#### 1.2 Initialize JSONL Log File `[AUTO]`

Ensure the log file exists so the tracker can append to it.

**Remote:**
```
touch ${OPENCLAW_CONFIG}/logs/model-usage.jsonl
```

Expected: File exists at `${OPENCLAW_CONFIG}/logs/model-usage.jsonl`. Verify:

**Remote:**
```
ls -la ${OPENCLAW_CONFIG}/logs/model-usage.jsonl
```

Expected: File listed with size (0 bytes for fresh install, or non-zero if pre-existing log data).

If this fails: Check file permissions on the `${OPENCLAW_CONFIG}/logs/` directory.

If already exists: `touch` updates the timestamp but preserves content. Existing log data is safe.

### Phase 2: Install the Skill

#### 2.1 Install Model Usage Tracker Skill `[GUIDED]`

Install the model-usage-tracker skill into the workspace. This creates the `log-usage.js` and `generate-report.js` scripts.

**Remote:**
```
openclaw skills install model-usage-tracker --path ${WORKSPACE}/skills/model-usage-tracker/
```

Expected: Skill installed successfully. Two scripts created in `${WORKSPACE}/skills/model-usage-tracker/`.

If this fails: Check that `openclaw` is on PATH and the workspace directory exists. If `openclaw skills install` is not available in the client's version, create the directory and scripts manually.

If already exists: Check if existing scripts are current. If they match, skip. If outdated, back up existing as `.bak` and reinstall.

#### 2.2 Verify Skill Files `[AUTO]`

Confirm the tracker scripts are in place.

**Remote:**
```
ls -la ${WORKSPACE}/skills/model-usage-tracker/
```

Expected: `log-usage.js` and `generate-report.js` listed.

If this fails: The skill install in step 2.1 did not complete. Re-run, or manually create the scripts.

### Phase 3: Test the Tracker

#### 3.1 Log a Test Entry `[AUTO]`

Write a test entry to verify the logging pipeline works end-to-end.

**Remote:**
```
cd ${WORKSPACE}/skills/model-usage-tracker && node log-usage.js test-model 100 50 test 'test entry'
```

Expected: No error output. A new line appended to the JSONL log.

Verify the entry was logged:

**Remote:**
```
tail -1 ${OPENCLAW_CONFIG}/logs/model-usage.jsonl
```

Expected: JSON object with `"model": "test-model"`, `"tokens_in": 100`, `"tokens_out": 50`, `"task_type": "test"`.

If this fails: Check that `node` is on PATH (requires Node >= 22). Check that the log path in `log-usage.js` matches `${OPENCLAW_CONFIG}/logs/model-usage.jsonl`. Check file write permissions.

#### 3.2 Generate a Test Report `[AUTO]`

Run the report generator to verify it can read and summarize the log.

**Remote:**
```
cd ${WORKSPACE}/skills/model-usage-tracker && node generate-report.js --days 7
```

Expected: Table output showing at least the test entry from step 3.1, with model name, call count, token counts, and estimated cost.

If this fails: Check that the JSONL log file path is correct in `generate-report.js`. Check that the test entry from step 3.1 was written successfully.

### Phase 4: Enable Agent Integration

#### 4.1 Enable Model Tracking in Agent Config `[GUIDED]`

Configure the OpenClaw agent middleware to call the usage tracker after every AI API call.

**Remote:**
```
openclaw config set model-tracking.enabled true
```

Expected: Configuration updated successfully.

If this fails: Check `openclaw config` command availability. If not supported, the integration may need to be configured manually in `${OPENCLAW_CONFIG}/openclaw.json`.

#### 4.2 Set Log Path in Agent Config `[AUTO]`

Point the agent middleware to the correct log file path.

**Remote:**
```
openclaw config set model-tracking.log-path '${OPENCLAW_CONFIG}/logs/model-usage.jsonl'
```

Expected: Configuration updated successfully.

If this fails: Same troubleshooting as step 4.1.

If already exists: If model tracking was previously configured, verify the existing path matches. Update only if different.

## Verification

Run these checks to confirm the model tracking system is operational.

Generate a 1-day report (should show at least the test entry):

**Remote:**
```
cd ${WORKSPACE}/skills/model-usage-tracker && node generate-report.js --days 1
```

Expected: Report table with at least the test entry from Phase 3.

After the agent runs real tasks, generate broader reports:

**Remote:**
```
cd ${WORKSPACE}/skills/model-usage-tracker && node generate-report.js --days 7
```

**Remote:**
```
cd ${WORKSPACE}/skills/model-usage-tracker && node generate-report.js --days 7 --group-by task-type
```

**Remote:**
```
cd ${WORKSPACE}/skills/model-usage-tracker && node generate-report.js --days 30 --group-by model
```

Expected:
- Report shows logged API calls with token counts and cost estimates
- Grouping by `task-type` shows which operations consume the most tokens
- Grouping by `model` shows cost distribution across providers

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| Phase 1: Set Up Log Storage | Execute silently | Execute silently | Confirm each command |
| Phase 2.1: Install Skill | Execute silently | Confirm before installing | Confirm each command |
| Phase 2.2: Verify Skill Files | Execute silently | Execute silently | Show output, confirm |
| Phase 3: Test the Tracker | Execute silently | Execute silently | Show output at each step |
| Phase 4.1: Enable Tracking | Execute silently | Confirm before enabling | Confirm each command |
| Phase 4.2: Set Log Path | Execute silently | Execute silently | Confirm |

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Empty reports | No API calls logged yet | Check if `log-usage.js` is being called by the agent middleware. Run a CRM query or other AI task and check the log. |
| Missing providers | Provider integration not calling `log-usage.js` | Each provider's API client needs to call `log-usage.js` after every call. Check the middleware configuration via `openclaw config get model-tracking`. |
| Cost estimates wrong | Token pricing outdated | Update the pricing table in `log-usage.js` to match current provider rates. Prices change frequently. |
| JSONL file growing too large | High API call volume over time | Rotate the log: `mv model-usage.jsonl model-usage.$(date +%Y%m).jsonl`. The report script reads from the current file. Consider adding log rotation to the backup script. |
| Duplicate entries | API retry logic logging the same call twice | Add dedup logic in the middleware (use request ID as dedup key). |
| Report shows $0 costs | Model not in pricing table | Add the model's pricing to the pricing config in `log-usage.js`. Unknown models default to $0. |
| `node` command not found | Node.js not on PATH | Refresh PATH or use full path to node. Check `openclaw doctor` for Node.js status. |

## Dependencies

- **Depends on:** OpenClaw agent installed (`openclaw/install.md`)
- **Independent of other systems** but should be deployed early to capture all costs from the start
- **Enhanced by:** `deploy-daily-briefing.md` (can include cost summary in morning briefing), `deploy-health-monitoring.md` (can flag anomalous spending)
