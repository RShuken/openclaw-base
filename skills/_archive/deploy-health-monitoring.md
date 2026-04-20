# Deploy Health Monitoring Heartbeat

## Purpose

Install a multi-cadence health monitoring system (daily/weekly/monthly checks). Philosophy: silent = healthy. Only alerts when something needs attention.

**When to use:** After the core workspace and key systems are running.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/memory/heartbeat-state.json 2>/dev/null"
cmd --session <id> --command "ls ${WORKSPACE}/tools/social-tracker/ 2>/dev/null"
cmd --session <id> --command "df -h / | tail -1"
```
- Is there an existing heartbeat state file?
- Which systems are deployed? (Determines which health checks are relevant)
- What's the current disk space situation?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Active checks | All 8 (social freshness, repo size, error logs, git backup, gateway binding, gateway auth, disk space, memory scan) | Subset of relevant checks | Not all checks apply. Skip social freshness if no social tracking. Skip gateway checks if no gateway. |
| Disk space thresholds | Warning at 20GB, urgent at 10GB | Adjust per disk size | Client has a smaller/larger disk |
| Repo size threshold | Alert at 500MB | Adjust | Large repos may need higher threshold |
| Social data freshness | Flag if older than 3 days | Adjust, or disable | Client's social tracking frequency |
| Check cadence | Daily/weekly/monthly per check type | Custom cadence | Client preference |
| Alert channel | Telegram (silent = healthy) | Discord, Slack, email | Client's messaging platform |
| Gateway checks | Verify localhost-only binding + auth | Skip if no gateway | Client doesn't run a gateway |
| Memory scan | Monthly prompt injection pattern scan | Disable if no memory files | Client doesn't use memory system |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram for alerts)
- `deploy-social-tracking.md` completed (for data freshness checks)
- `deploy-security-safety.md` completed (for security checks)
- Git configured (for backup checks)

## What Gets Installed

### Health Checks

| Cadence | Check | What |
|---------|-------|------|
| Daily | Social tracker freshness | Flag if data older than 3 days |
| Daily | Repo size | Alert if over 500MB (signals binary blob accumulation) |
| Daily | Error log scan | Recurring issues detection |
| Daily | Git backup | Workspace changes backed up |
| Weekly (Sunday 2am) | Gateway binding | Verify localhost-only, not exposed to internet |
| Weekly (Sunday 2am) | Gateway auth | Verify authentication is enabled |
| Weekly (Sunday 2am) | Disk space | Warning at 20GB free, urgent alert at 10GB |
| Monthly | Memory file scan | Check for prompt injection patterns |

### State File

`memory/heartbeat-state.json` -- tracks check timestamps so checks don't re-run unnecessarily.

```json
{
  "last_daily": "2026-02-23T07:00:00Z",
  "last_weekly": "2026-02-23T02:00:00Z",
  "last_monthly": "2026-02-01T02:00:00Z",
  "checks": {
    "social_freshness": { "last_run": "...", "status": "ok" },
    "repo_size": { "last_run": "...", "status": "ok", "size_mb": 142 },
    "error_log": { "last_run": "...", "status": "ok", "issues_found": 0 },
    "git_backup": { "last_run": "...", "status": "ok" },
    "gateway_binding": { "last_run": "...", "status": "ok" },
    "gateway_auth": { "last_run": "...", "status": "ok" },
    "disk_space": { "last_run": "...", "status": "ok", "free_gb": 45 },
    "memory_scan": { "last_run": "...", "status": "ok" }
  }
}
```

### Philosophy

Only alert when something needs attention. If the heartbeat system is silent, everything is fine. No daily "all clear" messages, no status reports unless requested. Alerts are actionable -- they tell you what's wrong and what to do.

### Scripts

| Script | Purpose |
|--------|---------|
| scripts/cron-health-check.sh | Every 30 min -- monitors cron jobs for error/timeout states |
| scripts/disk-space-check.sh | Weekly -- disk space monitoring with warning/urgent thresholds |
| scripts/security-review.sh | Nightly -- security baseline checks (gateway, auth, exposed ports) |

### Cron Jobs

| Job | Schedule |
|-----|----------|
| Cron Health Check | Every 30 min |
| Disk Space Check | Sunday 2am PST |

## Steps

### 1. Create Heartbeat State File

Initialize the state file in the memory directory.

```
cmd --session <id> --command "mkdir -p ~/clawd/memory"
cmd --session <id> --command "echo '{}' > ~/clawd/memory/heartbeat-state.json"
```

### 2. Install Health Check Scripts

Verify all scripts are in place and executable.

```
cmd --session <id> --command "ls -la ~/clawd/scripts/cron-health-check.sh ~/clawd/scripts/disk-space-check.sh ~/clawd/scripts/security-review.sh"
cmd --session <id> --command "chmod +x ~/clawd/scripts/cron-health-check.sh ~/clawd/scripts/disk-space-check.sh ~/clawd/scripts/security-review.sh"
```

### 3. Set Up Cron Health Check (Every 30 Min)

This is the core monitoring job that watches all other cron jobs for failures.

```
cmd --session <id> --command "openclaw cron add --name 'Cron Health Check' --schedule '*/30 * * * *' --command '~/clawd/scripts/cron-health-check.sh'"
```

### 4. Set Up Disk Space Check (Weekly Sunday 2am PST)

```
cmd --session <id> --command "openclaw cron add --name 'Disk Space Check' --schedule '0 2 * * 0' --tz 'America/Los_Angeles' --command '~/clawd/scripts/disk-space-check.sh'"
```

### 5. Configure Alert Thresholds

Set the thresholds for each check.

```
cmd --session <id> --command "openclaw config set health.social-freshness-days 3"
cmd --session <id> --command "openclaw config set health.repo-size-mb 500"
cmd --session <id> --command "openclaw config set health.disk-warning-gb 20"
cmd --session <id> --command "openclaw config set health.disk-urgent-gb 10"
```

### 6. Run Initial Health Check

Execute all health checks manually to establish a baseline.

```
cmd --session <id> --command "~/clawd/scripts/cron-health-check.sh"
cmd --session <id> --command "~/clawd/scripts/disk-space-check.sh"
```

Review the state file:

```
cmd --session <id> --command "cat ~/clawd/memory/heartbeat-state.json | python3 -m json.tool"
```

## Verification

Run these commands to confirm the health monitoring system is operational:

```
cmd --session <id> --command "~/clawd/scripts/cron-health-check.sh"
cmd --session <id> --command "~/clawd/scripts/disk-space-check.sh"
cmd --session <id> --command "cat ~/clawd/memory/heartbeat-state.json"
```

Expected output:
- Cron health check completes silently (no alerts = all healthy)
- Disk space check completes silently (if above thresholds)
- State file shows recent timestamps and "ok" status for all checks

Verify cron jobs are scheduled:

```
cmd --session <id> --command "openclaw cron list | grep -E 'Health|Disk'"
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| False alarms on data freshness | Social tracker cron jobs not scheduled or failing | Check: `openclaw cron list`. Verify social tracker is running. The 3-day threshold accounts for weekends. |
| Repo size alert | Accidentally committed binary files | Find large files: `git log --diff-filter=A --summary -- '*.db' '*.zip' '*.tar'`. Add to .gitignore. Consider BFG Repo-Cleaner for history rewriting. |
| Disk space alert (warning) | Log accumulation or old backups | Clean up: rotate logs, remove old backup archives. Check `du -sh ~/clawd/*` for largest directories. |
| Disk space alert (urgent) | Immediate action needed | Free space immediately: clean temp files, remove old docker images if applicable, rotate logs aggressively. |
| Gateway exposed to internet | Security configuration changed | Immediately reconfigure gateway to localhost-only binding. Run security-review.sh to verify. |
| Prompt injection detected | Memory files contain suspicious patterns | Review the flagged files manually. Do not auto-remediate -- human judgment required. |
| State file corrupted | Concurrent write or crash during update | Delete and re-initialize: `echo '{}' > ~/clawd/memory/heartbeat-state.json`. Next run will re-populate. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`
- **Enhanced by:** `deploy-social-tracking.md`, `deploy-security-safety.md`, `deploy-git-autosync.md`
