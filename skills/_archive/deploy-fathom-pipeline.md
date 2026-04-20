# Deploy Fathom Meeting Pipeline

## Purpose

Install an automated meeting processing pipeline. Polls Fathom for transcripts, matches attendees to CRM contacts, extracts action items with ownership (mine vs theirs), sends Telegram approval queue, and creates Todoist tasks for approved items.

**When to use:** When the user has Fathom recording meetings and wants automated action item tracking.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/crm/src/fathom/ 2>/dev/null"
cmd --session <id> --command "echo $FATHOM_API_KEY | head -c 8"
cmd --session <id> --command "which todoist 2>/dev/null"
cmd --session <id> --command "gog calendar events list --max-results 1 2>/dev/null"
```
- Does the client use Fathom for meeting recordings?
- Is the CRM deployed? (Required for contact matching)
- Is Todoist configured? (Required for task creation)
- Does the calendar integration work?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Meeting recorder | Fathom (API polling) | Otter.ai, Fireflies, Read.ai, manual transcripts | Client uses a different recording service |
| Task manager | Todoist (via todoist-ts-cli) | Linear, Asana, Notion, none | Client uses a different task tool |
| Calendar | Google Calendar via `gog` | Outlook, CalDAV | Client uses a different calendar |
| Polling schedule | Every 5 min, 7am-5pm PST | Adjust timezone and hours | Client's business hours and timezone |
| Completion check | 3x daily (8am, 12pm, 4pm PST) | Different times or frequency | Client's schedule |
| Approval channel | Telegram | Discord, Slack | Client's messaging platform |
| Auto-archive | 14 days | Longer/shorter | Client preference |
| Internal team filter | Exclude from "waiting on" | Custom domain list | Client's internal domains |

## Prerequisites

- `deploy-personal-crm.md` completed (contact matching)
- `deploy-google-workspace.md` completed (calendar awareness)
- `deploy-messaging-setup.md` completed (Telegram topic)
- Fathom API key (`FATHOM_API_KEY` in .env)
- Todoist configured (`TODOIST_API_TOKEN` in .env, `todoist-ts-cli` installed)

## What Gets Installed

### Components

| Component | File | Purpose |
|-----------|------|---------|
| API Client | crm/src/fathom/api-client.js | Fetch meetings, transcripts, summaries, action items from Fathom |
| Calendar | crm/src/fathom/calendar.js | Calendar integration -- fetches today's events, filters for likely Fathom meetings |
| After-Meetings Logic | crm/src/fathom/after-meetings-logic.js | Computes fire times (meeting end + buffer), determines when to poll |
| Processor | crm/src/fathom/processor.js | Match attendees to CRM, extract insights via Gemini Flash, create context entries, update summaries |
| Notifier | crm/src/fathom/notifier.js | Telegram notifications, approval queue for action items |
| Todoist Client | crm/src/fathom/todoist.js | Create Todoist tasks from approved items |

### Scripts

| Script | Purpose |
|--------|---------|
| fathom-sync.js | Main entry: --poll (new meetings) or --backfill (historical, 90 days default) |
| fathom-after-meetings.js | Calendar-aware polling -- checks if meetings ended, triggers after buffer (default 20 min) |
| fathom-approve.js | Approve verified action items, push to Todoist |

### Workflow

1. Every 5 min during business hours (7am-5pm PST), check if meetings ended
2. Wait for buffer period (meeting end + 20 min) before polling Fathom
3. Match attendees to CRM contacts by email
4. Extract action items: mine (create in Todoist after approval) vs theirs (track as "waiting on")
5. Send approval queue to Telegram
6. Completion check 3x daily (8am, 12pm, 4pm PST) -- shows overdue, pending, waiting-on
7. Auto-archive items older than 14 days
8. Exclude internal team members from "waiting on" list (only external contacts)

### Cron Jobs

| Job | Schedule |
|-----|----------|
| Fathom After-Meetings | Every 5 min, 7am-5pm PST |
| Action Item Completion Check | 8am, 12pm, 4pm PST |

## Steps

### 1. Set Fathom API Key

Add the Fathom API key to the environment configuration.

```
cmd --session <id> --command "openclaw config set-env FATHOM_API_KEY '<key>'"
```

Verify it was set:

```
cmd --session <id> --command "grep FATHOM_API_KEY ~/.openclaw/.env"
```

### 2. Install Todoist CLI

Install the Todoist TypeScript CLI globally and authenticate.

```
cmd --session <id> --command "npm install -g todoist-ts-cli && todoist auth <token>"
```

Verify the Todoist connection:

```
cmd --session <id> --command "todoist list --limit 1"
```

### 3. Install Fathom Modules

Verify all Fathom modules are present in the CRM source directory.

```
cmd --session <id> --command "ls -la ~/clawd/crm/src/fathom/"
```

Expected files: api-client.js, calendar.js, after-meetings-logic.js, processor.js, notifier.js, todoist.js

If any are missing, install them:

```
cmd --session <id> --command "cd ~/clawd/crm && npm install"
```

### 4. Run Initial Backfill

Backfill historical meeting data from Fathom (default: 90 days).

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/fathom-sync.js --backfill"
```

This may take several minutes depending on the number of historical meetings. The script will:
- Fetch all meetings from the last 90 days
- Match attendees to CRM contacts
- Create context entries for each meeting
- Extract and categorize action items

### 5. Set Up After-Meetings Cron

Schedule the calendar-aware polling job that checks for ended meetings every 5 minutes during business hours.

```
cmd --session <id> --command "openclaw cron add --name 'Fathom After-Meetings' --schedule '*/5 7-17 * * 1-5' --tz 'America/Los_Angeles' --command 'cd ~/clawd/crm && node scripts/fathom-after-meetings.js'"
```

### 6. Set Up Completion Check Cron

Schedule the 3x daily action item completion check.

```
cmd --session <id> --command "openclaw cron add --name 'Action Item Completion Check' --schedule '0 8,12,16 * * 1-5' --tz 'America/Los_Angeles' --command 'cd ~/clawd/crm && node scripts/fathom-approve.js --check'"
```

### 7. Test Approval Flow

Run a manual poll and verify the Telegram approval flow works end-to-end.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/fathom-sync.js --poll"
cmd --session <id> --command "cd ~/clawd/crm && node scripts/fathom-approve.js --list"
```

If action items appear, test approving one:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/fathom-approve.js --approve <item-id>"
```

Verify the Todoist task was created:

```
cmd --session <id> --command "todoist list --limit 5"
```

## Verification

Run these commands to confirm the Fathom pipeline is fully operational:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/fathom-sync.js --poll"
cmd --session <id> --command "cd ~/clawd/crm && node scripts/fathom-approve.js --list"
```

Expected output:
- Poll should complete without errors (may report 0 new meetings if none recently ended)
- Approve list should show any pending action items with ownership labels (mine/theirs)
- Telegram should receive notifications for any new action items found

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No transcripts found | FATHOM_API_KEY invalid or meetings not recorded | Verify the API key: `curl -H "Authorization: Bearer $FATHOM_API_KEY" https://api.fathom.video/v1/meetings`. Check that Fathom is recording meetings. |
| Attendees not matching CRM | Contacts not synced or email mismatch | Run CRM sync first: `cd ~/clawd/crm && node scripts/sync.js`. Verify contact emails match Fathom attendee emails. |
| Calendar not detecting meetings | Google Calendar access broken | Verify gog calendar access: `gog calendar list --today`. Re-authenticate if needed. |
| Todoist tasks not created | TODOIST_API_TOKEN invalid or CLI not authed | Verify token: `todoist list --limit 1`. Re-authenticate: `todoist auth <token>`. |
| Buffer timing wrong | After-meetings logic computing wrong fire times | Check after-meetings-logic.js buffer constant (default 20 min). Inspect cron logs for timing. |
| Action items all "theirs" | Ownership detection confused | Check processor.js ownership logic. Verify the user's email is configured so "mine" items are correctly identified. |
| Approval queue empty | No new meetings or all items already processed | Run `--backfill` to reprocess historical data. Check if meetings have action items in Fathom UI. |

## Dependencies

- **Requires:** `deploy-personal-crm.md`, `deploy-google-workspace.md`, `deploy-messaging-setup.md`
- **Feeds into:** `deploy-daily-briefing.md`, `deploy-advisory-council.md`
