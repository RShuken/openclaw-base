# Deploy Model Usage & Cost Tracking

## Purpose

Install a usage tracker that logs every AI API call across all providers. Tracks model, tokens, task type, and estimated cost. Generates reports to monitor spending.

**When to use:** Any time AI APIs are being used. Deploy early to track costs from the start.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${OPENCLAW_CONFIG}/logs/model-usage.jsonl 2>/dev/null"
cmd --session <id> --command "openclaw config get models"
```
- Is there an existing usage log? (Upgrade vs fresh install)
- Which AI providers does the client use?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Log format | JSONL at `${OPENCLAW_CONFIG}/logs/model-usage.jsonl` | SQLite, custom path | Client wants queryable database instead of flat file |
| Providers tracked | All (Anthropic, OpenAI, Google, xAI) | Subset | Client only uses some providers |
| Cost estimation | Built-in token pricing | Custom pricing, or skip cost tracking | Client has negotiated rates or subscription pricing |
| Report delivery | CLI on demand | Automated daily/weekly to Telegram/Slack | Client wants proactive cost alerts |
| Cost alerts | None (manual review) | Alert when daily/weekly spend exceeds threshold | Client wants spend guardrails |

## Prerequisites

- OpenClaw agent installed
- At least one AI provider configured

## What Gets Installed

### Model Usage Tracker Skill (`skills/model-usage-tracker/`)

| Script | Purpose |
|--------|---------|
| log-usage.js | Log a single API call: model, tokens_in, tokens_out, task_type, description |
| generate-report.js | Generate cost reports with filters: --days N, --model X, --task-type Y |

### Storage

JSONL log at `~/.openclaw/logs/model-usage.jsonl`

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
| timestamp | ISO 8601 timestamp of the API call |
| model | Provider/model identifier (e.g., anthropic/claude-opus-4-6, google/gemini-2.0-flash) |
| tokens_in | Input tokens consumed |
| tokens_out | Output tokens generated |
| task_type | Category: conversation, embedding, classification, research, summarization, code |
| description | Brief description of what the call was for |
| estimated_cost | USD cost estimate based on current provider pricing |

### Reports

The report generator supports multiple output formats and filters.

| Flag | Purpose |
|------|---------|
| --days N | Show last N days (default: 7) |
| --model X | Filter by model name or provider |
| --task-type Y | Filter by task type |
| --format table\|json\|csv | Output format (default: table) |
| --group-by model\|task-type\|day | Group results (default: model) |

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

### Self-Monitoring

The system tracks its own expenses so the user always knows how much it costs to run. Every AI call made by the agent (CRM queries, email classification, meeting processing, research) is logged.

## Steps

### 1. Create Log Directory

Ensure the logs directory exists.

```
cmd --session <id> --command "mkdir -p ~/.openclaw/logs"
```

### 2. Install Model Usage Tracker Skill

```
cmd --session <id> --command "openclaw skill install model-usage-tracker --path ~/clawd/skills/model-usage-tracker/"
```

Verify the scripts are in place:

```
cmd --session <id> --command "ls -la ~/clawd/skills/model-usage-tracker/"
```

Expected files: log-usage.js, generate-report.js

### 3. Verify JSONL Log Path

```
cmd --session <id> --command "touch ~/.openclaw/logs/model-usage.jsonl && ls -la ~/.openclaw/logs/model-usage.jsonl"
```

### 4. Log a Test Entry

```
cmd --session <id> --command "cd ~/clawd/skills/model-usage-tracker && node log-usage.js test-model 100 50 test 'test entry'"
```

Verify the entry was logged:

```
cmd --session <id> --command "tail -1 ~/.openclaw/logs/model-usage.jsonl"
```

### 5. Generate a Test Report

```
cmd --session <id> --command "cd ~/clawd/skills/model-usage-tracker && node generate-report.js --days 7"
```

Should show at least the test entry from Step 4.

### 6. Integrate with Agent Middleware

The usage tracker needs to be called after every AI API call. This is typically done via middleware in the agent's API client.

```
cmd --session <id> --command "openclaw config set model-tracking.enabled true"
cmd --session <id> --command "openclaw config set model-tracking.log-path '~/.openclaw/logs/model-usage.jsonl'"
```

## Verification

Run these commands to confirm the model tracking system is operational:

```
cd ~/clawd/skills/model-usage-tracker
node generate-report.js --days 1
```

Should show at least the test entry. After the agent runs real tasks:

```
cd ~/clawd/skills/model-usage-tracker
node generate-report.js --days 7
node generate-report.js --days 7 --group-by task-type
node generate-report.js --days 30 --group-by model
```

Expected output:
- Report shows logged API calls with token counts and cost estimates
- Grouping by task-type shows which operations consume the most tokens
- Grouping by model shows cost distribution across providers

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Empty reports | No API calls logged yet | Check if log-usage.js is being called by the agent middleware. Run a CRM query or other AI task and check the log. |
| Missing providers | Provider integration not calling log-usage.js | Each provider's API client needs to call log-usage.js after every call. Check the middleware configuration. |
| Cost estimates wrong | Token pricing outdated | Update the pricing table in log-usage.js to match current provider rates. Prices change frequently. |
| JSONL file growing too large | High API call volume over time | Rotate the log: `mv model-usage.jsonl model-usage.$(date +%Y%m).jsonl`. The report script reads from the current file. Consider adding log rotation to the backup script. |
| Duplicate entries | API retry logic logging the same call twice | Add dedup logic in the middleware (use request ID as dedup key). |
| Report shows $0 costs | Model not in pricing table | Add the model's pricing to the pricing config in log-usage.js. Unknown models default to $0. |

## Dependencies

- **Requires:** OpenClaw agent installed
- **Independent of other systems** (but should be deployed early to capture all costs from the start)
