---
name: "Deploy Fathom Meeting Pipeline"
category: "integration"
subcategory: "meeting-intelligence"
third_party_service: "Fathom"
auth_type: "api-key"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "FATHOM_EXTRACTION_MODEL"
clawhub_alternative: "fathom-api"
cost_tier: "api-usage"
privacy_tier: "cloud"
requires:
  - "Fathom API key"
  - "Todoist API token"
  - "Google Calendar access (via gog)"
  - "Telegram channel configured"
docs_urls:
  - "https://docs.fathom.ai/api-reference"
  - "https://developer.todoist.com/rest/v2/"
---

# Deploy Fathom Meeting Pipeline

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING (Fathom-specific)
- **Previously-flagged commands, reality-checked:**
  - `openclaw cron add/list/rm` — all available on 2026.4.15 per `audit/_baseline.md` §3 (note: `rm` not `remove`)
  - `openclaw config set-env` — never existed. For env vars, write directly to `~/.openclaw/.env` (mode 600) which the gateway reads; or use `openclaw config set` for scalar in-config values
- **Scheduling**: `openclaw cron add --name fathom-ingest --cron "*/15 * * * *"` for poll-based Fathom ingestion
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`FATHOM_EXTRACTION_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to action-item extraction from meeting transcripts.
- **Fathom is one transcript source**. If the client uses a different recorder (Granola, Otter, Zoom AI Companion, Gemini Meeting Notes), swap the source layer — the CRM-matching + action-item layers stay identical. See ClawHub alternatives under §B: `fathom-api`, `fathom-meetings`.

## Purpose

Install an automated meeting processing pipeline that bridges Fathom recordings with CRM contact matching, AI-driven action item extraction, Telegram approval workflows, and Todoist task creation. This is a multi-system integration: it connects Fathom (meeting source), the CRM (contact matching), Google Calendar (meeting schedule awareness), Telegram (approval UI), and Todoist (task destination) into a single automated flow.

**When to use:** When the client has Fathom recording meetings and wants automated action item tracking with CRM-aware context and approval-gated task creation.

**What this skill does:**
1. Configures the Fathom API key and Todoist CLI for the pipeline endpoints
2. Verifies all Fathom pipeline modules are present in the CRM source directory
3. Runs a historical backfill of meeting data (default 90 days)
4. Schedules calendar-aware polling (every 5 min during business hours)
5. Schedules 3x daily action item completion checks
6. Tests the end-to-end approval flow (poll, list, approve, Todoist task creation)

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${FATHOM_API_KEY}` | Client-provided Fathom API key | `fth_k3y...` |
| `${TODOIST_API_TOKEN}` | Client-provided Todoist API token | `abc123def456` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${BIZ_HOURS_START}` | Client preference: business hours start (24h) | `7` |
| `${BIZ_HOURS_END}` | Client preference: business hours end (24h) | `17` |
| `${TELEGRAM_TOPIC_ID}` | Created during messaging setup, or client profile | `1207` |
| `${CLIENT_EMAIL}` | Client profile: primary email (for "mine" vs "theirs" ownership) | `user@example.com` |
| `${INTERNAL_DOMAINS}` | Client profile: internal team email domains to exclude from "waiting on" | `company.com,team.co` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the Fathom pipeline modules already exist:

**Remote:**
```
ls ${WORKSPACE}/crm/src/fathom/ 2>/dev/null || echo 'NOT_FOUND'
```

Check if the Fathom API key is already configured:

**Remote:**
```
grep FATHOM_API_KEY ${WORKSPACE}/.env 2>/dev/null | head -c 20 || echo 'NOT_SET'
```

Check if Todoist CLI is installed:

**Remote:**
```
which todoist 2>/dev/null || echo 'NOT_INSTALLED'
```

Check if Google Calendar integration works (required dependency):

**Remote:**
```
gog calendar events list --max-results 1 2>/dev/null && echo 'CALENDAR_OK' || echo 'CALENDAR_FAIL'
```

Check if CRM is deployed (required dependency):

**Remote:**
```
ls ${WORKSPACE}/crm/src/ 2>/dev/null && echo 'CRM_OK' || echo 'CRM_MISSING'
```

**Decision points from pre-flight:**
- Is the CRM deployed? **Required.** If not, deploy `deploy-personal-crm.md` first.
- Is Google Calendar working? **Required.** If not, deploy `deploy-google-workspace.md` first.
- Is messaging set up? **Required** for approval flow. If not, deploy `deploy-messaging-setup.md` first.
- Is the Fathom pipeline already partially deployed? Check for existing cron jobs and decide whether to update or skip.
- Does the client use Todoist, or a different task manager? Adapt Step 2 and the todoist module accordingly.

## Adaptation Points

The defaults below work for most installs. Cross-reference against the client profile for overrides.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Meeting recorder | Fathom (API polling) | Client uses Otter.ai, Fireflies, Read.ai, or manual transcripts |
| Task manager | Todoist via `todoist-ts-cli` | Client uses Linear, Asana, Notion, or no task manager |
| Calendar | Google Calendar via `gog` | Client uses Outlook, CalDAV, or another calendar |
| Polling schedule | Every 5 min, 7am-5pm Mon-Fri | Client's timezone or business hours differ |
| Completion check | 3x daily (8am, 12pm, 4pm) | Client wants different times or frequency |
| Approval channel | Telegram topic | Client uses Discord, Slack, or another messaging platform |
| Post-meeting buffer | 20 minutes after meeting ends | Client wants faster or slower processing |
| Auto-archive | 14 days | Client wants longer or shorter retention |
| Internal team filter | Exclude from "waiting on" | Client needs custom domain list for internal contacts |

## Prerequisites

- `deploy-personal-crm.md` completed (contact matching for attendees)
- `deploy-google-workspace.md` completed (calendar awareness for meeting detection)
- `deploy-messaging-setup.md` completed (Telegram topic for approval queue)
- Fathom API key (client provides this; obtained from Fathom dashboard)
- Todoist API token (client provides this; obtained from Todoist settings > Integrations)

## What Gets Installed

### Components

| Component | File | Purpose |
|-----------|------|---------|
| API Client | `crm/src/fathom/api-client.js` | Fetch meetings, transcripts, summaries, action items from Fathom |
| Calendar | `crm/src/fathom/calendar.js` | Calendar integration -- fetches today's events, filters for likely Fathom meetings |
| After-Meetings Logic | `crm/src/fathom/after-meetings-logic.js` | Computes fire times (meeting end + buffer), determines when to poll |
| Processor | `crm/src/fathom/processor.js` | Match attendees to CRM, extract insights via Gemini Flash, create context entries, update summaries |
| Notifier | `crm/src/fathom/notifier.js` | Telegram notifications, approval queue for action items |
| Todoist Client | `crm/src/fathom/todoist.js` | Create Todoist tasks from approved items |

### Scripts

| Script | Purpose |
|--------|---------|
| `fathom-sync.js` | Main entry: `--poll` (new meetings) or `--backfill` (historical, 90 days default) |
| `fathom-after-meetings.js` | Calendar-aware polling -- checks if meetings ended, triggers after buffer (default 20 min) |
| `fathom-approve.js` | Approve verified action items, push to Todoist |

### Workflow

1. Every 5 min during business hours, check if meetings ended (calendar-aware)
2. Wait for buffer period (meeting end + 20 min) before polling Fathom
3. Match attendees to CRM contacts by email
4. Extract action items: mine (create in Todoist after approval) vs theirs (track as "waiting on")
5. Send approval queue to Telegram topic
6. Completion check 3x daily -- shows overdue, pending, waiting-on
7. Auto-archive items older than 14 days
8. Exclude internal team members from "waiting on" list (only external contacts)

### Cron Jobs

| Job | Schedule | Description |
|-----|----------|-------------|
| Fathom After-Meetings | `*/5 ${BIZ_HOURS_START}-${BIZ_HOURS_END} * * 1-5` in `${TIMEZONE}` | Calendar-aware meeting poll |
| Action Item Completion Check | `0 8,12,16 * * 1-5` in `${TIMEZONE}` | Review overdue and pending items |

## Steps

### Phase 1: Configure API Credentials `[HUMAN_INPUT]`

#### 1.1 Set Fathom API Key

Store the Fathom API key in the workspace environment so the pipeline scripts can access it.
The client must provide this key (from Fathom dashboard > Settings > API).

**Remote:**
```
openclaw config set-env FATHOM_API_KEY '${FATHOM_API_KEY}'
```

Expected: The environment variable is stored. Verify with a quick grep.

**Remote:**
```
grep FATHOM_API_KEY ${WORKSPACE}/.env | head -c 20
```

Expected: First 20 characters of the line confirm the key is set (never log the full key).

If this fails: Check that `openclaw config set-env` is available. If not, manually append to the `.env` file: `echo "FATHOM_API_KEY=${FATHOM_API_KEY}" >> ${WORKSPACE}/.env`.

If already exists: Compare the first few characters. If the key matches, skip. If different, ask the client whether to update it.

#### 1.2 Install and Authenticate Todoist CLI

Install the Todoist TypeScript CLI globally and authenticate it with the client's API token.
This provides the `todoist` command used by the approval flow to create tasks.

**Note:** The exact authentication syntax for `todoist-ts-cli` may vary by version. The intent is to install the CLI globally and then authenticate it using the client's Todoist API token. Check `todoist --help` or the package README for the current auth command (it may be `todoist auth <token>`, `todoist login`, or require setting a `TODOIST_API_TOKEN` environment variable).

Install the CLI globally:

**Remote:**
```
npm install -g todoist-ts-cli
```

Expected: The `todoist` command becomes available on PATH.

If this fails: Check Node.js and npm are installed. Try `npm install -g todoist-ts-cli@latest`.

Authenticate with the Todoist API token (verify the exact syntax first):

**Remote:**
```
todoist --help
```

Based on the help output, run the appropriate auth command. Common patterns:
- `todoist auth ${TODOIST_API_TOKEN}`
- Setting `TODOIST_API_TOKEN` in the environment via `openclaw config set-env`

Verify the Todoist connection by listing tasks:

**Remote:**
```
todoist list --limit 1
```

Expected: At least one task listed, or an empty list with no errors. This confirms auth is working.

If this fails: The API token may be invalid. Ask the client to verify it from Todoist Settings > Integrations > Developer. Also check if the CLI needs a different auth flow.

If already exists: Run `todoist list --limit 1` to verify the existing auth works. If it does, skip reinstall.

### Phase 2: Verify Pipeline Modules `[GUIDED]`

#### 2.1 Check Fathom Modules

Verify all six Fathom pipeline modules are present in the CRM source directory.
These modules should have been deployed as part of the CRM or a prior pipeline setup.

**Remote:**
```
ls -la ${WORKSPACE}/crm/src/fathom/
```

Expected: All six files present: `api-client.js`, `calendar.js`, `after-meetings-logic.js`, `processor.js`, `notifier.js`, `todoist.js`.

If any modules are missing: They may need to be created or the CRM npm dependencies may need updating.

**Remote:**
```
cd ${WORKSPACE}/crm && npm install
```

If this fails: Check that the CRM was deployed via `deploy-personal-crm.md`. The Fathom modules live inside the CRM project. If the CRM is not deployed, stop and deploy it first.

If already exists: All modules present -- skip to Phase 3.

### Phase 3: Historical Backfill `[GUIDED]`

#### 3.1 Run Initial Backfill

Backfill historical meeting data from Fathom (default: 90 days). This populates the CRM with meeting context so the pipeline has history to work from.

This step may take several minutes depending on meeting volume. The script fetches all meetings, matches attendees to CRM contacts, creates context entries, and categorizes action items.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/fathom-sync.js --backfill
```

Expected: Script completes with a summary of meetings processed, contacts matched, and action items extracted. No errors.

If this fails:
- **"FATHOM_API_KEY not set"**: Go back to Step 1.1.
- **"Cannot connect to Fathom API"**: Verify the API key is valid. Test directly: `curl -sS -H "Authorization: Bearer ${FATHOM_API_KEY}" https://api.fathom.video/v1/meetings | head -c 200`.
- **CRM database errors**: The CRM database may need migration. Run `cd ${WORKSPACE}/crm && node scripts/migrate.js`.
- **Timeout on large history**: For clients with many meetings, use `--backfill --days 30` for a shorter window first, then expand.

Idempotency: Safe to re-run. The backfill script checks for already-processed meetings and skips them.

### Phase 4: Schedule Cron Jobs `[GUIDED]`

#### 4.1 Set Up After-Meetings Cron

Schedule the calendar-aware polling job. This runs every 5 minutes during business hours on weekdays. It checks Google Calendar for recently-ended meetings, waits for the post-meeting buffer (20 min default), then polls Fathom for transcripts and action items.

**Remote:**
```
openclaw cron add --name 'Fathom After-Meetings' --schedule '*/5 ${BIZ_HOURS_START}-${BIZ_HOURS_END} * * 1-5' --tz '${TIMEZONE}' \
  --env FATHOM_EXTRACTION_MODEL=openai-codex/gpt-5.4-nano \
  --command 'cd ${WORKSPACE}/crm && node scripts/fathom-after-meetings.js'
```

`FATHOM_EXTRACTION_MODEL` routes the action-item extraction LLM call. Omit to defer to the agent default. In `scripts/fathom-after-meetings.js`:

```javascript
const MODEL = process.env.FATHOM_EXTRACTION_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

Expected: Cron job registered. Verify with `openclaw cron list`.

If this fails: Check that `openclaw cron add` is available. If using system crontab instead, adapt to `crontab -e` format.

If already exists: Check `openclaw cron list` for a duplicate. If the schedule matches, skip. If different, remove the old one first with `openclaw cron rm --name 'Fathom After-Meetings'` then re-add.

#### 4.2 Set Up Completion Check Cron

Schedule the 3x daily action item review. This checks for overdue, pending, and "waiting on" items and sends a summary to the Telegram topic.

**Remote:**
```
openclaw cron add --name 'Action Item Completion Check' --schedule '0 8,12,16 * * 1-5' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/crm && node scripts/fathom-approve.js --check'
```

Expected: Cron job registered. Verify with `openclaw cron list`.

If this fails: Same as 4.1 -- check `openclaw cron` availability.

If already exists: Check for duplicate. Update schedule if the client's preferred times differ.

### Phase 5: Test End-to-End Flow `[GUIDED]`

#### 5.1 Run Manual Poll

Trigger a manual meeting poll to verify the pipeline processes meetings correctly.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/fathom-sync.js --poll
```

Expected: Script completes without errors. If meetings recently ended, it should find and process them. If no recent meetings, it reports "0 new meetings" -- this is normal.

If this fails:
- **Fathom API error**: Verify API key (Step 1.1).
- **Calendar error**: Verify Google Calendar access: `gog calendar events list --max-results 1`.
- **CRM matching error**: Verify CRM is healthy: `cd ${WORKSPACE}/crm && node scripts/sync.js --status`.

#### 5.2 List Pending Action Items

Check the approval queue to see if action items were extracted.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/fathom-approve.js --list
```

Expected: Lists pending action items with ownership labels ("mine" vs "theirs"). If no recent meetings with action items, the list may be empty.

#### 5.3 Test Approval Flow (if items exist)

If action items appeared in Step 5.2, test approving one to verify the Todoist integration.

Approve a specific item (replace `ITEM_ID` with an actual ID from the list):

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/fathom-approve.js --approve ITEM_ID
```

Expected: Item marked as approved. If it was a "mine" item, a Todoist task should be created.

Verify the Todoist task was created:

**Remote:**
```
todoist list --limit 5
```

Expected: The approved action item appears as a new Todoist task.

If this fails:
- **Todoist auth error**: Re-run the auth step (Phase 1, Step 1.2).
- **"No item with ID"**: The item ID may have changed. Re-run `--list` to get current IDs.

#### 5.4 Verify Telegram Notifications

Check that the approval queue notification was sent to the configured Telegram topic. This is a manual verification -- ask the client to check their Telegram group for the notification, or check the agent logs.

**Remote:**
```
tail -20 ${WORKSPACE}/crm/logs/fathom.log 2>/dev/null || echo 'NO_LOG_FILE'
```

Expected: Log entries showing Telegram notification sent successfully.

If this fails: Check Telegram topic configuration. Verify `deploy-messaging-setup.md` was completed and the Fathom topic exists.

## Verification

Run these checks to confirm the full pipeline is operational:

**Cron jobs registered:**

**Remote:**
```
openclaw cron list
```

Expected: Both "Fathom After-Meetings" and "Action Item Completion Check" appear with correct schedules.

**Pipeline processes meetings:**

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/fathom-sync.js --poll
```

Expected: Completes without errors (0 meetings is fine if none recently ended).

**Approval queue accessible:**

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/fathom-approve.js --list
```

Expected: Lists pending items or empty list with no errors.

**All integration points responding:**

| Integration | Check Command | Expected |
|-------------|---------------|----------|
| Fathom API | `curl -sS -H "Authorization: Bearer ${FATHOM_API_KEY}" https://api.fathom.video/v1/meetings \| head -c 100` | JSON response with meetings array |
| Google Calendar | `gog calendar events list --max-results 1` | Calendar event listed or empty list |
| Todoist | `todoist list --limit 1` | Task listed or empty list, no auth errors |
| CRM database | `cd ${WORKSPACE}/crm && node scripts/sync.js --status` | Database stats, no errors |
| Telegram | Check agent logs for recent notification delivery | Sent successfully |

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No transcripts found | `FATHOM_API_KEY` invalid or meetings not recorded | Verify the API key: `curl -sS -H "Authorization: Bearer ${FATHOM_API_KEY}" https://api.fathom.video/v1/meetings \| head -c 200`. Check that Fathom is recording meetings. |
| Attendees not matching CRM | Contacts not synced or email mismatch | Run CRM sync: `cd ${WORKSPACE}/crm && node scripts/sync.js`. Verify contact emails match Fathom attendee emails. |
| Calendar not detecting meetings | Google Calendar access broken | Verify: `gog calendar events list --max-results 3`. Re-authenticate via `deploy-google-workspace.md` if needed. |
| Todoist tasks not created | API token invalid or CLI not authenticated | Verify: `todoist list --limit 1`. Re-authenticate (see Phase 1, Step 1.2). |
| Buffer timing wrong | After-meetings logic computing wrong fire times | Check `after-meetings-logic.js` buffer constant (default 20 min). Inspect cron logs for timing. |
| Action items all "theirs" | Ownership detection confused | Check `processor.js` ownership logic. Verify `${CLIENT_EMAIL}` is configured so "mine" items are correctly identified. |
| Approval queue always empty | No new meetings or all items already processed | Run `--backfill --days 7` to reprocess recent data. Check if meetings have action items in Fathom UI. |
| Cron job not firing | Timezone or schedule misconfigured | Check `openclaw cron list` for schedule. Verify timezone matches client's actual timezone. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| Phase 1: Configure API Credentials | Prompt for keys only | Prompt for keys, confirm before setting | Prompt for keys, confirm each command |
| Phase 2: Verify Pipeline Modules | Execute silently | Confirm before npm install | Confirm each check |
| Phase 3: Historical Backfill | Execute silently | Confirm before backfill (may take minutes) | Confirm, show progress |
| Phase 4: Schedule Cron Jobs | Execute silently | Confirm schedule and timezone | Confirm each cron job |
| Phase 5: Test End-to-End Flow | Execute silently, report results | Confirm before each test step | Confirm each command, review output |
| Approval test (5.3) | Skip if no items | Offer to test if items exist | Walk through approval step by step |

**Note:** Phase 1 is always `[HUMAN_INPUT]` regardless of autonomy mode -- API keys require the client to provide them.

## Dependencies

- **Depends on:**
  - `deploy-personal-crm.md` -- CRM must exist for contact matching (attendee email lookup)
  - `deploy-google-workspace.md` -- Calendar must work for meeting schedule awareness
  - `deploy-messaging-setup.md` -- Telegram topic must exist for approval queue notifications
- **Required by:**
  - `deploy-daily-briefing.md` -- Briefing includes action item status from this pipeline
  - `deploy-advisory-council.md` -- Advisory council uses meeting context from this pipeline
- **Integration dependency chain:** Fathom API -> Calendar (meeting detection) -> CRM (contact matching) -> Processor (AI extraction) -> Notifier (Telegram approval) -> Todoist (task creation). If any link breaks, downstream steps will fail. Debug from left to right.
