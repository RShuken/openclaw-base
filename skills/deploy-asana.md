---
name: "Deploy Asana Integration"
category: "integration"
subcategory: "productivity"
third_party_service: "Asana"
auth_type: "personal-access-token"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "asana-api"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Asana Personal Access Token"
  - "SQLite"
  - "openclaw cron (2026.4.15+)"
docs_urls:
  - "https://developers.asana.com/docs/personal-access-token"
  - "https://developers.asana.com/reference/rest-api-reference"
---

# Deploy Asana Integration

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale; `openclaw cron add/list` both exist on 2026.4.15 per `audit/_baseline.md` §3
- **Scheduling**: `openclaw cron add --name asana-sync --cron "*/15 * * * *"` (native, replaces launchd/crontab/schtasks fallback)
- **Model**: agent default (Codex-first); do not hardcode
- **ClawHub alternatives**: `asana-api` (managed-OAuth), `asana-pat`, `asana-agent-skill` per §B — evaluate before scratch-building

## Purpose

Connect the agent to Asana for project management. Syncs tasks from configured projects into a local SQLite database, serves as the destination for video pipeline cards, and feeds task/status data into the business advisory council's analysis.

**When to use:** When the client uses Asana for project management and wants task sync, action items briefing, or video pipeline integration.

**What this skill does:**
1. Configures Asana API credentials
2. Installs the Asana fetch tool and sync script
3. Runs an initial sync to populate the local database
4. Schedules recurring sync via cron
5. Installs the action items briefing script
6. Verifies end-to-end data flow

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${ASANA_PAT}` | Client: Asana > My Settings > Apps > Personal Access Tokens | `1/12345678:abcdef...` |
| `${ASANA_WORKSPACE_ID}` | Asana API: `GET /api/1.0/workspaces` or client provides | `1200000000000000` |
| `${ASANA_PROJECT_IDS}` | Client provides; comma-separated list of project GIDs | `1212455754265217,1212346778656423` |
| `${VIDEO_PIPELINE_PROJECT_ID}` | Client provides: the Asana project for video idea cards | `1212455754265217` |
| `${SYNC_SCHEDULE}` | Client preference (cron expression) | `0 */4 * * *` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the client has an Asana PAT configured:

**Remote:**
```
grep ASANA_PAT ${WORKSPACE}/.env 2>/dev/null | head -c 20 || echo 'NOT_CONFIGURED'
```

Check if the Asana fetch tool already exists:

**Remote:**
```
test -f ${WORKSPACE}/tools/asana-fetch.js && echo 'EXISTS' || echo 'NOT_FOUND'
```

Check if a sync database already exists:

**Remote:**
```
test -f ${WORKSPACE}/data/asana-sync.db && echo 'EXISTS' || echo 'NOT_FOUND'
```

Check if the sync cron is already scheduled:

**Remote:**
```
openclaw cron list 2>/dev/null | grep -i asana || echo 'NO_CRON'
```

**Decision points from pre-flight:**
- Does the client use Asana? If not, this skill does not apply.
- Which Asana workspace and projects should be synced? Get project GIDs from the client.
- Is the video pipeline going to use Asana as its destination?
- Is the advisory council deployed? If so, Asana data will feed into it automatically after sync.

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Project management tool | Asana | Client uses Linear, Jira, Notion, Trello, or GitHub Issues instead |
| Project IDs | Client-specific (must be provided) | Every client will have different projects |
| Sync frequency | Every 4 hours (`0 */4 * * *`) | Client wants more or less frequent sync |
| Update rule | Add as comments, don't edit descriptions | Client prefers direct description edits |
| Video pipeline destination | `${VIDEO_PIPELINE_PROJECT_ID}` | Client uses a different project or a different tool entirely |
| Action items sections | OVERDUE / ACTION ITEMS / WAITING ON | Client wants custom section names or groupings |
| Sync database path | `${WORKSPACE}/data/asana-sync.db` | Client wants databases in a different location |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed
- Asana account with Personal Access Token
- Asana workspace ID and project IDs known

## What Gets Installed

### Asana Fetch Tool (`tools/asana-fetch.js`)

| Capability | Description |
|------------|-------------|
| Fetch tasks | Pull tasks from Asana projects by GID |
| Task details | Task name, completion status, due dates, assignees |
| Section grouping | Tasks organized by project sections |
| Notes and tags | Full task notes and tag metadata |

### Asana Sync (`tools/business-meta-analysis/sync/asana-sync.cjs`)

| Detail | Value |
|--------|-------|
| Data synced | Task name, assignee, notes, tags, section, completion status |
| Database | `${WORKSPACE}/data/asana-sync.db` |
| Schedule | Every 4 hours (configurable via `${SYNC_SCHEDULE}`) |

### Action Items Briefing (`scripts/action-items-briefing.js`)

Reads from `asana-sync.db` for daily prep. Generates a structured briefing:

| Section | Content |
|---------|---------|
| OVERDUE | Tasks past their due date |
| ACTION ITEMS | Tasks due today or upcoming that need attention |
| WAITING ON | Tasks assigned to others that are blocking progress |

### Update Rule

When updating tasks, add new information as comments rather than editing the description. This preserves history and makes it easy to see what changed and when.

## Steps

### Phase 1: Credentials and Directory Setup

#### 1.1 Configure Asana PAT `[HUMAN_INPUT]`

Ensure the Asana Personal Access Token is in the workspace `.env` file. The client must provide their PAT from Asana > My Settings > Apps > Personal Access Tokens.

**Remote:**
```
grep ASANA_PAT ${WORKSPACE}/.env 2>/dev/null || echo 'NOT_CONFIGURED'
```

If not configured, append it:

**Remote:**
```
echo 'ASANA_PAT=${ASANA_PAT}' >> ${WORKSPACE}/.env
```

Expected: `grep ASANA_PAT ${WORKSPACE}/.env` returns the token line (first ~20 chars visible for verification without exposing the full secret).

If this fails: Verify the `.env` file exists and is writable. If the workspace hasn't been initialized yet, create the file.

If already exists: Check that the token is valid (not expired). If it's correct, skip this step.

#### 1.2 Create Data Directory `[AUTO]`

Ensure the data directory exists for the sync database.

**Remote:**
```
mkdir -p ${WORKSPACE}/data
```

Expected: Directory exists at `${WORKSPACE}/data/`.

If already exists: `mkdir -p` is idempotent -- skip silently.

#### 1.3 Create Tools Directory Structure `[AUTO]`

Ensure the directory structure for the sync script exists.

**Remote:**
```
mkdir -p ${WORKSPACE}/tools/business-meta-analysis/sync
```

Expected: Full directory path exists.

If already exists: `mkdir -p` is idempotent -- skip silently.

### Phase 2: Install Scripts

#### 2.1 Install Asana Fetch Tool `[GUIDED]`

Write the Asana fetch tool that pulls tasks from configured projects. This tool is used both for on-demand task queries and by other scripts.

The fetch tool should:
- Read `ASANA_PAT` from the workspace `.env` file
- Accept project GIDs as arguments or use configured defaults
- Output task details (name, due date, assignee, section, completion status, notes, tags)
- Support a `--list-projects` flag to enumerate workspace projects

Write the tool to `${WORKSPACE}/tools/asana-fetch.js`.

**Remote:**
```
cat > ${WORKSPACE}/tools/asana-fetch.js << 'EOF'
[asana-fetch.js content - the operator should use the existing implementation
from the client's codebase or write one that matches the capabilities above]
EOF
```

Expected: File exists at `${WORKSPACE}/tools/asana-fetch.js` and is executable with `node`.

If this fails: Check that Node.js is available. Verify the tools directory exists (Phase 1.3).

If already exists: Compare content. If unchanged, skip. If different, back up as `asana-fetch.js.bak` and write new version.

#### 2.2 Install Asana Sync Script `[GUIDED]`

Write the sync script that pulls tasks from configured Asana projects into a local SQLite database. This runs on a schedule and keeps the local database current.

The sync script should:
- Read `ASANA_PAT` from the workspace `.env` file
- Sync tasks from the configured project GIDs (`${ASANA_PROJECT_IDS}`)
- Store results in `${WORKSPACE}/data/asana-sync.db`
- Support a `--status` flag to show last sync time and task counts
- Handle pagination for large projects (1000+ tasks)

Write the script to `${WORKSPACE}/tools/business-meta-analysis/sync/asana-sync.cjs`.

**Remote:**
```
cat > ${WORKSPACE}/tools/business-meta-analysis/sync/asana-sync.cjs << 'EOF'
[asana-sync.cjs content - the operator should use the existing implementation
or write one that matches the capabilities above]
EOF
```

Expected: File exists at `${WORKSPACE}/tools/business-meta-analysis/sync/asana-sync.cjs`.

If this fails: Check directory exists (Phase 1.3). Check write permissions.

If already exists: Compare content. If unchanged, skip. If different, back up as `asana-sync.cjs.bak` and write new version.

#### 2.3 Install Action Items Briefing Script `[GUIDED]`

Write the action items briefing script that reads from the sync database and generates a structured task summary. This is called by the daily briefing system.

The briefing script should:
- Read from `${WORKSPACE}/data/asana-sync.db`
- Generate three sections: OVERDUE, ACTION ITEMS, WAITING ON
- OVERDUE: tasks past their due date
- ACTION ITEMS: tasks due today or upcoming that need attention
- WAITING ON: tasks assigned to others that are blocking progress
- Output formatted text suitable for messaging (Telegram/Slack)

Write the script to `${WORKSPACE}/scripts/action-items-briefing.js`.

**Remote:**
```
cat > ${WORKSPACE}/scripts/action-items-briefing.js << 'EOF'
[action-items-briefing.js content - the operator should use the existing
implementation or write one matching the capabilities above]
EOF
```

Expected: File exists at `${WORKSPACE}/scripts/action-items-briefing.js`.

If this fails: Ensure `${WORKSPACE}/scripts/` directory exists. Create with `mkdir -p ${WORKSPACE}/scripts`.

If already exists: Compare content. If unchanged, skip. If different, back up and write new version.

### Phase 3: Initial Sync and Scheduling

#### 3.1 Run Initial Sync `[GUIDED]`

Run the sync script to populate the local database with current task data from Asana.

**Remote:**
```
cd ${WORKSPACE} && node tools/business-meta-analysis/sync/asana-sync.cjs
```

Expected: Script completes without errors. Creates/populates `${WORKSPACE}/data/asana-sync.db` with tasks from the configured projects.

If this fails:
- **Auth failure (401):** The `ASANA_PAT` is invalid or expired. Generate a new token in Asana > My Settings > Apps > Personal Access Tokens and update `.env`.
- **No tasks returned:** Wrong workspace or project IDs. Verify project GIDs match the client's Asana workspace. Use the fetch tool with `--list-projects` to enumerate available projects.
- **Network error:** Check internet connectivity on the client machine.
- **SQLite error:** Check that the `${WORKSPACE}/data/` directory exists and is writable.

#### 3.2 Schedule Recurring Sync `[AUTO]`

Set up a cron job to sync Asana data at the configured interval.

**Remote:**
```
openclaw cron add --name 'Asana Sync' --schedule '${SYNC_SCHEDULE}' --tz '${TIMEZONE}' --command 'node ${WORKSPACE}/tools/business-meta-analysis/sync/asana-sync.cjs'
```

Expected: Cron job registered. Verify with `openclaw cron list | grep -i asana`.

If this fails: Check that `openclaw cron` is available. If not, fall back to system crontab: add the entry via `crontab -e` (macOS/Linux) or `schtasks` (Windows).

If already exists: Check if the existing schedule matches. If it does, skip. If different, update the existing cron job rather than creating a duplicate.

## Verification

Run these checks to confirm the Asana integration is working end-to-end.

### Verify fetch tool works

**Remote:**
```
cd ${WORKSPACE} && node tools/asana-fetch.js
```

Expected: Tasks listed from configured projects with names, due dates, assignees, and sections.

### Verify sync database has data

**Remote:**
```
cd ${WORKSPACE} && node tools/business-meta-analysis/sync/asana-sync.cjs --status
```

Expected: Last sync timestamp shown, task count per project, no error messages.

### Verify action items briefing

**Remote:**
```
cd ${WORKSPACE} && node scripts/action-items-briefing.js
```

Expected: Output includes OVERDUE, ACTION ITEMS, and WAITING ON sections (some may be empty if no matching tasks exist).

### Verify cron is scheduled

**Remote:**
```
openclaw cron list | grep -i asana
```

Expected: Shows the Asana Sync job with the configured schedule.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Auth failure (401) | `ASANA_PAT` is invalid or expired | Generate a new token in Asana > My Settings > Apps > Personal Access Tokens. Update `${WORKSPACE}/.env`. |
| No tasks returned | Wrong workspace or project IDs | Verify project GIDs match the client's Asana workspace. Use fetch tool with `--list-projects` if available. |
| Sync cron not running | Cron job misconfigured | Check `openclaw cron list` and cron logs for errors. |
| Stale data after sync | Pagination issue on large projects | Check sync logs. Projects with 1000+ tasks may need the pagination limit increased in the sync script. |
| Action items briefing empty | No tasks in sync database | Run sync first, then briefing. The briefing reads from `asana-sync.db`, not the API directly. |
| Comments not appearing on tasks | API permissions | Ensure `ASANA_PAT` has write access. Personal Access Tokens get full access by default, but check if the workspace has restrictions. |
| Video pipeline cards not syncing | Project ID mismatch | Verify `${VIDEO_PIPELINE_PROJECT_ID}` matches the current Asana project. If the project was recreated, the GID will have changed. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Configure Asana PAT | Pause for `[HUMAN_INPUT]` -- client must provide token | Pause for `[HUMAN_INPUT]` | Pause for `[HUMAN_INPUT]` |
| 1.2 Create data directory | Execute silently | Execute silently | Confirm before creating |
| 1.3 Create tools directory | Execute silently | Execute silently | Confirm before creating |
| 2.1 Install fetch tool | Execute silently | Confirm before writing | Confirm before writing |
| 2.2 Install sync script | Execute silently | Confirm before writing | Confirm before writing |
| 2.3 Install briefing script | Execute silently | Confirm before writing | Confirm before writing |
| 3.1 Run initial sync | Execute silently | Confirm before running | Confirm, show output |
| 3.2 Schedule recurring sync | Execute silently | Confirm schedule | Confirm schedule and command |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed), `deploy-messaging-setup.md` (for notification delivery)
- **Required by:** `deploy-video-pipeline.md` (Video Pipeline project is the destination for video idea cards)
- **Feeds into:** `deploy-advisory-council.md` (TeamDynamicsArchitect and AutomationScout personas use Asana data), `deploy-daily-briefing.md` (action items briefing section)
- **Independent of:** `deploy-newsletter-crm.md`, `deploy-social-tracking.md` (but all enhance the advisory council together)
