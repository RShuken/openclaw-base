# Deploy Daily Morning Briefing

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale. `openclaw cron` (add/list/edit) and `openclaw config set` all exist on 2026.4.15 per `audit/_baseline.md` §3/§4C.7 (note: `cron update` is `cron edit`)
- **Scheduling**: `openclaw cron add --name daily-briefing --cron "0 7 * * *" --tz ${TIMEZONE}`
- **Config**: `openclaw config set` for scalar briefing settings; for `briefing.*` namespace, put values in workspace files (`BRIEFING.md` or `TOOLS.md`) — OpenClaw reads those natively
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`BRIEFING_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples).

## Purpose

Install a comprehensive 7am daily briefing delivered to Telegram. Combines calendar with CRM context, yesterday's content performance, pending action items, and email cross-references into a single consolidated message.

**When to use:** After CRM and social tracking are deployed. The more data sources, the richer the briefing.

**What this skill does:**
1. Verifies data sources (CRM, calendar, social tracking) are populated
2. Configures the daily briefing cron job for the client's preferred time
3. Routes the briefing to the correct Telegram topic
4. Runs a test briefing to verify all sections populate

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${BRIEFING_TOPIC_ID}` | Client profile: messaging config (Meeting Prep topic) | `1964` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${BRIEFING_TIME}` | Client preference (default: 7am weekdays) | `0 7 * * 1-5` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**

Check if the CRM database exists (required for attendee context):

**Remote:**
```
ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null || echo 'NO_CRM'
```

Expected: File path printed. If `NO_CRM`, deploy `deploy-personal-crm.md` first.

Check if calendar access is working:

**Remote:**
```
gog calendar events list --max-results 1 2>/dev/null || echo 'NO_CALENDAR'
```

Expected: At least one event returned, or empty list (both OK). If `NO_CALENDAR`, deploy `deploy-google-workspace.md` first.

Check which optional data sources are available:

**Remote:**
```
ls ${WORKSPACE}/scripts/social-stats.js 2>/dev/null && echo 'SOCIAL_TRACKING_OK'
ls ${WORKSPACE}/scripts/fathom-sync.js 2>/dev/null && echo 'FATHOM_OK'
```

Expected: Note which are available — they enrich the briefing but aren't required.

**Decision points from pre-flight:**
- What time does the client want the briefing? (Default: 7am weekdays in their timezone)
- Which optional data sources are available? (Social tracking, Fathom, etc.)
- Which Telegram topic should receive the briefing?

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Delivery time | 7am weekdays in `${TIMEZONE}` | Client has different morning routine |
| Delivery channel | Telegram Meeting Prep topic | Client uses Discord, Slack, or email |
| Calendar source | Google Calendar via `gog` | Client uses Outlook or CalDAV |
| Attendee context | Full CRM lookup per attendee | CRM not deployed, or client wants lighter briefing |
| Content performance | YouTube + Instagram + X metrics | Only include platforms the client tracks |
| Action items | Fathom meeting action items | Fathom not deployed — section omitted gracefully |
| Email cross-reference | Gmail threads related to meetings | Email integration not deployed |
| Briefing sections | All (calendar, content, actions, waiting-on, email) | Client wants a simpler/shorter briefing |

## Prerequisites

- `deploy-personal-crm.md` completed (attendee context)
- `deploy-google-workspace.md` completed (calendar, email)
- `deploy-messaging-setup.md` completed (Telegram Meeting Prep topic)
- Recommended: `deploy-social-tracking.md` (content performance data)
- Recommended: `deploy-fathom-pipeline.md` (action items)

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
- Single consolidated Telegram message to Meeting Prep topic
- Keeps urgent emails, CRM notifications, follow-ups in their own separate topics (no duplication)
- Smart truncation for Telegram message limits (prioritizes today's meetings over historical data)
- Graceful degradation: missing data sources result in omitted sections, not errors

### Cron Jobs

| Job | Schedule |
|-----|----------|
| Daily Brief | Weekdays at `${BRIEFING_TIME}` in `${TIMEZONE}` |

## Steps

### Phase 1: Verify Data Sources

#### 1.1 Confirm CRM Data is Populated `[AUTO]`

Ensure the CRM has contacts loaded so the briefing can look up meeting attendees.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/query.js 'stats'
```

Expected: Contact count > 0. Shows number of contacts, companies, and interactions.

If this fails: Run CRM sync first: `cd ${WORKSPACE}/crm && node scripts/sync.js`. If the script doesn't exist, the CRM may need initial population — see `deploy-personal-crm.md`.

#### 1.2 Confirm Calendar Access `[AUTO]`

Verify the agent can read today's calendar events.

**Remote:**
```
gog calendar list --today
```

Expected: List of today's events (or empty if no meetings scheduled). Command exits cleanly.

If this fails: Re-authenticate calendar access. See `deploy-google-workspace.md` for `gog` OAuth refresh.

#### 1.3 Check Optional Data Sources `[AUTO]`

If social tracking is deployed, verify it has recent data.

**Remote:**
```
cd ${WORKSPACE} && node scripts/social-stats.js --yesterday
```

Expected: Yesterday's metrics for tracked platforms. If this script doesn't exist, social tracking isn't deployed — the briefing will omit the content performance section gracefully.

### Phase 2: Configure the Briefing

#### 2.1 Schedule the Daily Briefing Cron Job `[GUIDED]`

Schedule the briefing to run at the client's preferred time on weekdays.

**Remote:**
```
openclaw cron add --name 'Daily Brief' --schedule '${BRIEFING_TIME}' --tz '${TIMEZONE}' \
  --env BRIEFING_MODEL=openai-codex/gpt-5.4-nano \
  --command 'cd ${WORKSPACE} && node scripts/daily-brief.js'
```

`BRIEFING_MODEL` routes the morning synthesis LLM call to a specific model. Omit to defer to the agent default. In `scripts/daily-brief.js`:

```javascript
const MODEL = process.env.BRIEFING_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

Expected: Cron job created successfully. Confirmation message with schedule details.

If this fails: Check if `openclaw cron` is available. If not, use the system crontab directly: add the entry via `crontab -e` with the appropriate schedule and `TZ=${TIMEZONE}` prefix.

If already exists: Check the existing schedule matches the desired time. If different, update with `openclaw cron edit --name 'Daily Brief' --schedule '${BRIEFING_TIME}'`. If the same, skip.

#### 2.2 Set Telegram Briefing Topic `[GUIDED]`

Route the daily briefing to the Meeting Prep topic in Telegram.

**Remote:**
```
openclaw config set briefing.telegram-topic ${BRIEFING_TOPIC_ID}
```

Expected: Config updated confirmation.

If this fails: Check that `openclaw config` is available. If not, edit the config file directly at `${WORKSPACE}/config/briefing.json` or equivalent.

If already exists: Verify the current topic ID matches `${BRIEFING_TOPIC_ID}`. If it does, skip.

### Phase 3: Test and Verify

#### 3.1 Run a Manual Test Briefing `[GUIDED]`

Trigger the briefing manually to verify all sections populate correctly.

**Remote:**
```
cd ${WORKSPACE} && node scripts/daily-brief.js
```

Expected: Briefing message appears in the Telegram Meeting Prep topic. Check that:
- Calendar section shows today's meetings with attendee context
- Content performance shows yesterday's metrics (if social tracking deployed)
- Action items show pending items (if Fathom pipeline deployed)
- Email cross-reference shows related threads (if email integration deployed)

If this fails: Check the script output for errors. Common issues: missing environment variables, API auth expired, Telegram bot token not set.

#### 3.2 Verify Cron Job is Registered `[AUTO]`

Confirm the cron job is scheduled and will fire at the right time.

**Remote:**
```
openclaw cron list | grep 'Daily Brief'
```

Expected: Shows the Daily Brief cron job with the correct schedule and timezone.

If this fails: Re-run Step 2.1 to register the cron job.

## Verification

After deployment, verify the full system:

**Remote:**
```
cd ${WORKSPACE} && node scripts/daily-brief.js
```

Check the Telegram Meeting Prep topic for the briefing message.

**Remote:**
```
openclaw cron list | grep 'Daily Brief'
```

Expected:
- Calendar section shows CRM context per attendee (name, company, last interaction, relationship score)
- Content performance numbers are from yesterday (if social tracking deployed)
- Action items reflect current state (if Fathom pipeline deployed)
- Briefing fits within Telegram message limits
- Cron job is scheduled for the correct time and timezone

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Missing attendee context | CRM contacts not synced | Run CRM sync: `cd ${WORKSPACE}/crm && node scripts/sync.js`. Ensure attendee emails match CRM contact emails. |
| No content performance | Social tracking not deployed or cron not running | Deploy `deploy-social-tracking.md` first. Check social tracker cron jobs are active. |
| Empty action items section | Fathom pipeline not deployed | Deploy `deploy-fathom-pipeline.md` for meeting action items. Without it, this section is omitted gracefully. |
| Briefing too long | Too many meetings or action items | System auto-truncates for Telegram limits. If consistently too long, review priority settings in daily-brief.js. |
| Calendar shows no meetings | Google Calendar access broken | Verify gog calendar access: `gog calendar list --today`. Re-authenticate if needed. |
| Briefing not sending at scheduled time | Cron job misconfigured or timezone wrong | Verify cron schedule: `openclaw cron list`. Check timezone matches `${TIMEZONE}`. |
| Email cross-reference empty | No email threads match today's meeting attendees | Normal if today's meetings are with contacts who don't have recent email threads. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1-1.3 Verify data sources | Execute silently | Execute silently | Show output, confirm |
| 2.1 Schedule cron job | Execute silently | Confirm schedule and timezone | Confirm command |
| 2.2 Set Telegram topic | Execute silently | Confirm topic ID | Confirm command |
| 3.1 Run test briefing | Execute, report result | Execute, review output together | Confirm, review together |
| 3.2 Verify cron | Execute silently | Execute silently | Show output |

## Dependencies

- **Requires:** `deploy-personal-crm.md`, `deploy-google-workspace.md`, `deploy-messaging-setup.md`
- **Enhanced by:** `deploy-social-tracking.md`, `deploy-fathom-pipeline.md`
- **Required by:** None (this is a consumer of other systems' data)
