# Deploy Daily Morning Briefing

## Purpose

Install a comprehensive 7am daily briefing delivered to Telegram. Combines calendar with CRM context, yesterday's content performance, pending action items, and email cross-references into a single consolidated message.

**When to use:** After CRM and social tracking are deployed. The more data sources, the richer the briefing.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null"
cmd --session <id> --command "gog calendar events list --max-results 1 2>/dev/null"
```
- Is the CRM deployed? (Required for attendee context)
- Is calendar access working? (Core of the briefing)
- Which optional data sources are available? (Social tracking, Fathom, etc.)
- What time does the client want the briefing?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Delivery time | 7am PST weekdays | Any time, any days, client timezone | Client's morning routine and timezone |
| Delivery channel | Telegram Meeting Prep topic | Discord, Slack, email | Client's messaging platform |
| Calendar source | Google Calendar via `gog` | Outlook, CalDAV | Client's calendar provider |
| Attendee context | Full CRM lookup per attendee | Basic calendar info only | CRM not deployed, or client wants lighter briefing |
| Content performance | YouTube + Instagram + X metrics | Subset of platforms, or skip | Only include platforms the client tracks |
| Action items | Fathom meeting action items | Manual task list, or skip | Fathom not deployed |
| Email cross-reference | Gmail threads related to meetings | Skip if no email integration | Email integration not deployed |
| Briefing sections | All (calendar, content, actions, waiting-on, email) | Subset | Client wants a simpler/shorter briefing |

## Prerequisites

- `deploy-personal-crm.md` completed (attendee context)
- `deploy-google-workspace.md` completed (calendar, email)
- `deploy-messaging-setup.md` completed (Telegram Meeting Prep topic)
- Recommended: `deploy-social-tracking.md` completed (content performance data)
- Recommended: `deploy-fathom-pipeline.md` completed (action items)

## What Gets Installed

### Daily Brief Contents

| Section | What's Included |
|---------|----------------|
| Calendar | Today's meetings with full CRM context per attendee: who they are, company, last discussion, relevant history |
| Content Performance | Yesterday's YouTube/Instagram/X metrics: views, engagement, outliers |
| Action Items | Pending from meetings, what's overdue |
| Waiting On | Items waiting on other people |
| Email Cross-Reference | Email threads related to today's meetings |

### Features

- Full CRM context on every meeting attendee (not just "meeting with Greg" but who Greg is, what company, what you discussed last time, their relationship score)
- Optional background research on important meeting participants
- Single consolidated Telegram message (Meeting Prep topic, ID: 1964)
- Keeps urgent emails, CRM notifications, follow-ups in their own separate topics (no duplication)
- Smart truncation for Telegram message limits (prioritizes today's meetings over historical data)
- Graceful degradation: missing data sources result in omitted sections, not errors

### Cron Jobs

| Job | Schedule |
|-----|----------|
| Daily Brief | Weekdays 7am PST |

## Steps

### 1. Verify Data Sources Are Populated

Ensure the CRM, calendar, and any optional data sources have data available for the briefing.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/query.js 'stats'"
cmd --session <id> --command "gog calendar list --today"
```

If social tracking is deployed:

```
cmd --session <id> --command "cd ~/clawd && node scripts/social-stats.js --yesterday"
```

### 2. Configure Daily Briefing Cron Job

Schedule the briefing to run at 7am PST on weekdays.

```
cmd --session <id> --command "openclaw cron add --name 'Daily Brief' --schedule '0 7 * * 1-5' --tz 'America/Los_Angeles' --command 'cd ~/clawd && node scripts/daily-brief.js'"
```

### 3. Set Telegram Meeting Prep Topic

Ensure the briefing routes to the Meeting Prep topic (ID: 1964).

```
cmd --session <id> --command "openclaw config set briefing.telegram-topic 1964"
```

### 4. Run a Manual Test Briefing

Trigger the briefing manually to verify all sections populate correctly.

```
cmd --session <id> --command "cd ~/clawd && node scripts/daily-brief.js"
```

Check the Telegram Meeting Prep topic for the briefing message.

### 5. Verify All Sections Populated

Review the briefing output and confirm:
- Calendar section shows today's meetings with attendee context
- Content performance shows yesterday's metrics (if social tracking deployed)
- Action items show pending items (if Fathom pipeline deployed)
- Email cross-reference shows related threads

If any section is empty, check the corresponding data source.

## Verification

Run these commands to confirm the daily briefing system is operational:

```
cmd --session <id> --command "cd ~/clawd && node scripts/daily-brief.js"
```

Check the Telegram Meeting Prep topic for the briefing message. Verify:

```
cmd --session <id> --command "openclaw cron list | grep 'Daily Brief'"
```

Expected output:
- Calendar section shows CRM context per attendee (name, company, last interaction, relationship score)
- Content performance numbers are from yesterday (if social tracking deployed)
- Action items reflect current state (if Fathom pipeline deployed)
- Briefing fits within Telegram message limits
- Cron job is scheduled for 7am PST weekdays

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Missing attendee context | CRM contacts not synced | Run CRM sync: `cd ~/clawd/crm && node scripts/sync.js`. Ensure attendee emails match CRM contact emails. |
| No content performance | Social tracking not deployed or cron not running | Deploy `deploy-social-tracking.md` first. Check social tracker cron jobs are active. |
| Empty action items section | Fathom pipeline not deployed | Deploy `deploy-fathom-pipeline.md` for meeting action items. Without it, this section is omitted gracefully. |
| Briefing too long | Too many meetings or action items | System auto-truncates for Telegram message limits. If consistently too long, review the priority settings in daily-brief.js. |
| Calendar shows no meetings | Google Calendar access broken | Verify gog calendar access: `gog calendar list --today`. Re-authenticate if needed. |
| Briefing not sending at 7am | Cron job misconfigured or timezone wrong | Verify cron schedule: `openclaw cron list`. Check timezone is set to America/Los_Angeles. |
| Email cross-reference empty | No email threads match today's meeting attendees | This is normal if today's meetings are with contacts who don't have recent email threads. |

## Dependencies

- **Requires:** `deploy-personal-crm.md`, `deploy-google-workspace.md`, `deploy-messaging-setup.md`
- **Enhanced by:** `deploy-social-tracking.md`, `deploy-fathom-pipeline.md`
