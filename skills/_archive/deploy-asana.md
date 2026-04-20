# Deploy Asana Integration

## Purpose

Connect the agent to Asana for project management. Syncs tasks, serves as destination for video pipeline cards, and feeds status data into the business advisory council.

**When to use:** When the user uses Asana for project management.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "echo $ASANA_PAT | head -c 8"
cmd --session <id> --command "node ${WORKSPACE}/tools/asana-fetch.js 2>/dev/null | head -5"
```
- Does the client use Asana?
- Which Asana workspace and projects should be synced?
- Is the video pipeline going to use Asana as its destination?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Project management tool | Asana | Linear, Jira, Notion, Trello, GitHub Issues | Client uses a different PM tool |
| Project IDs | Matt's workspace IDs (Video Pipeline, Ad Bank, etc.) | Client's own project IDs | Every client will have different projects |
| Sync frequency | Every 4 hours | More/less frequent | Client preference |
| Update rule | Add as comments, don't edit descriptions | Edit descriptions directly | Client's preference for task history |
| Video pipeline destination | Asana Video Pipeline project | Different project, or different tool entirely | Client's workflow |
| Action items briefing | OVERDUE / ACTION ITEMS / WAITING ON sections | Custom sections | Client's task tracking needs |

## Prerequisites

- `deploy-messaging-setup.md` completed
- Asana account with Personal Access Token (`ASANA_PAT` in `.env`)
- Asana workspace ID known

## What Gets Installed

### Asana Fetch Tool (`tools/asana-fetch.js`)

| Capability | Description |
|------------|-------------|
| Fetch tasks | Pull tasks from Asana projects |
| Task details | Shows task name, completion status, due dates, assignees |
| Section grouping | Tasks organized by project sections |
| Notes and tags | Full task notes and tag metadata |

### Asana Sync (`tools/business-meta-analysis/sync/`)

| Data | Details |
|------|---------|
| Tasks | Name, assignee, notes, tags, section, completion status |
| Database | `~/clawd/data/asana-sync.db` |
| Schedule | Every 4 hours |

### Action Items Briefing (`scripts/action-items-briefing.js`)

Reads from `asana-sync.db` for daily prep. Generates a structured briefing with three sections:

| Section | Content |
|---------|---------|
| OVERDUE | Tasks past their due date |
| ACTION ITEMS | Tasks due today or upcoming that need attention |
| WAITING ON | Tasks assigned to others that are blocking progress |

### Key Projects

| Project | ID | Purpose |
|---------|-----|---------|
| Video Pipeline | 1212455754265217 | Destination for video idea cards |
| The Ad Bank / Sponsorship Queue | 1212346778656423 | Sponsorship tracking |
| FF Team Meeting | 1209311439712808 | Team meeting items |

### Update Rule

When updating tasks, add new information as comments rather than editing the description. This preserves history and makes it easy to see what changed and when.

## Steps

### 1. Configure Asana Credentials

```
cmd --session <id> --command "grep ASANA_PAT ~/clawd/.env"
```

If missing:

```
cmd --session <id> --command "echo 'ASANA_PAT=<your-personal-access-token>' >> ~/clawd/.env"
```

Get a Personal Access Token from: Asana > My Settings > Apps > Personal Access Tokens.

### 2. Install Asana Fetch Tool

```
cmd --session <id> --command "cp /path/to/asana-fetch.js ~/clawd/tools/asana-fetch.js"
```

### 3. Install Asana Sync Script

```
cmd --session <id> --command "cp /path/to/asana-sync.cjs ~/clawd/tools/business-meta-analysis/sync/asana-sync.cjs"
```

### 4. Create Data Directory

```
cmd --session <id> --command "mkdir -p ~/clawd/data"
```

### 5. Run Initial Sync

```
cmd --session <id> --command "node ~/clawd/tools/business-meta-analysis/sync/asana-sync.cjs"
```

Expected: Pulls tasks from configured projects into `~/clawd/data/asana-sync.db`.

### 6. Set Up 4-Hour Sync Cron Job

```
cmd --session <id> --command "openclaw cron add --name 'Asana Sync' --schedule '0 */4 * * *' --tz 'America/Los_Angeles' --command 'node ~/clawd/tools/business-meta-analysis/sync/asana-sync.cjs'"
```

### 7. Test Asana Fetch

```
cmd --session <id> --command "node ~/clawd/tools/asana-fetch.js"
```

Expected: Lists tasks from configured projects with task names, due dates, and assignees.

### 8. Test Action Items Briefing

```
cmd --session <id> --command "node ~/clawd/scripts/action-items-briefing.js"
```

Expected: Outputs OVERDUE, ACTION ITEMS, and WAITING ON sections based on current task data.

## Verification

Run these commands to confirm the Asana integration is working:

```
node ~/clawd/tools/asana-fetch.js
```

Expected:
- Tasks listed from configured projects
- Task details include names, due dates, assignees, sections

```
node ~/clawd/tools/business-meta-analysis/sync/asana-sync.cjs --status
```

Expected:
- Last sync timestamp
- Task count per project
- No error messages

```
node ~/clawd/scripts/action-items-briefing.js
```

Expected:
- OVERDUE section (may be empty if nothing is overdue)
- ACTION ITEMS section with upcoming tasks
- WAITING ON section with delegated tasks

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Auth failure (401) | `ASANA_PAT` is invalid or expired | Generate a new token in Asana > My Settings > Apps > Personal Access Tokens. Update `.env`. |
| No tasks returned | Wrong workspace or project IDs | Verify project IDs match your Asana workspace. Check with `node ~/clawd/tools/asana-fetch.js --list-projects` if available. |
| Sync cron not running | Cron job misconfigured | Check `openclaw cron list` and cron-log for errors. |
| Stale data after sync | Pagination issue on large projects | Check sync logs. Large projects with 1000+ tasks may need the pagination limit increased in the sync script. |
| Action items briefing empty | No tasks in sync database | Run sync first, then briefing. The briefing reads from `asana-sync.db`, not the API directly. |
| Comments not appearing on tasks | API permissions | Ensure `ASANA_PAT` has write access. Personal Access Tokens get full access by default, but check if the workspace has restrictions. |
| Video pipeline cards not syncing | Project ID mismatch | Verify Video Pipeline project ID is `1212455754265217`. If the project was recreated, the ID will have changed. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`
- **Required by:** `deploy-video-pipeline.md` (Video Pipeline project is the destination for video idea cards)
- **Feeds into:** `deploy-advisory-council.md` (TeamDynamicsArchitect and AutomationScout personas use Asana data)
- **Independent of:** `deploy-newsletter-crm.md`, `deploy-social-tracking.md` (but all enhance the advisory council together)
