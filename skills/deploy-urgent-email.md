# Deploy Urgent Email Detection

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" was stale. `openclaw config get/set` and `openclaw cron` (list/edit) all exist on 2026.4.15 per `audit/_baseline.md` §3/§4C.7 (note: `cron update` is `cron edit`)
- **Scheduling**: `openclaw cron add --name urgent-email-scan --cron "*/5 * * * *"` (poll every 5 min — or reduce to 15 min to save cost)
- **Config**: `openclaw config set` for scalar thresholds; classification prompts go in workspace files the skill reads
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`URGENT_EMAIL_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Cheap/haiku-class model recommended for per-email classification (e.g., `openai-codex/gpt-5.4-nano`).
- **ClawHub alternatives**: `email-triage`, `expanso-email-triage`, `email-triage-pro` per §B

## Purpose

Install an AI-powered email scanner that detects urgent emails and alerts via Telegram. Includes a feedback learning loop — learns from "was/wasn't urgent" corrections to improve over time.

**When to use:** When the client wants proactive alerts for important emails without constant inbox monitoring.

**What this skill does:**
1. Ensures the CRM database has the `urgent_notifications` table
2. Verifies urgent email scripts are in place
3. Chains the urgent check as a post-hook on the Email Refresh cron job
4. Configures alert routing to the correct Telegram topic
5. Tests the scanner and feedback loop

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CRM_TOPIC_ID}` | Client profile: messaging config (CRM topic) | `709` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**

Check if the CRM database exists:

**Remote:**
```
ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null || echo 'NO_CRM'
```

Expected: File path printed. If `NO_CRM`, deploy `deploy-personal-crm.md` first.

Check if Gmail access is working:

**Remote:**
```
gog gmail messages list --max-results 1 2>/dev/null || echo 'NO_GMAIL'
```

Expected: At least one message returned. If `NO_GMAIL`, deploy `deploy-google-workspace.md` first.

**Decision points from pre-flight:**
- What are the client's waking hours? (Default time gates may not fit their schedule)
- What Telegram topic should receive urgent alerts?
- Does the Email Refresh cron job already exist? (The urgent check hooks onto it)

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Email source | Gmail via `gog` CLI | Client uses Microsoft 365 or IMAP |
| Alert time window | Weekdays 5-9pm, weekends 7am-9pm in `${TIMEZONE}` | Client has different waking hours |
| Scan frequency | Every 30 min during waking hours (via Email Refresh post-hook) | Client wants more/less frequent scans |
| Alert channel | Telegram CRM topic | Client uses Discord, Slack, or SMS |
| Noise filter | Pre-filter known noise senders | Client has different noise patterns |
| Feedback mechanism | "Was/wasn't urgent" replies in Telegram | Client prefers CLI feedback or different channel |

## Prerequisites

- `deploy-personal-crm.md` completed (CRM database for notification tracking)
- `deploy-google-workspace.md` completed (Gmail access)
- `deploy-messaging-setup.md` completed (Telegram CRM topic)

## What Gets Installed

### Scripts

| Script | Purpose |
|--------|---------|
| `urgent-email-check.js` | Scan recently-ingested inbound emails for urgency |
| `urgent-feedback.js` | Process feedback on notifications (useful/noise) |

### Features

- AI classification with few-shot learning from feedback
- Time-gated alerts: weekdays 5-9pm, weekends 7am-9pm in `${TIMEZONE}` (no middle-of-night alerts)
- Pre-filters noise senders (newsletters, marketing, automated)
- Dedicated Telegram topic for urgent-only alerts
- Feedback loop: reply "was/wasn't urgent" → LLM extracts verdict and reasoning → improves future classification
- Notification dedup (won't alert twice for same email)

### Storage

`urgent_notifications` table in CRM database (`contacts.db`):

| Column | Purpose |
|--------|---------|
| id | Primary key |
| email_id | Gmail message ID (dedup key) |
| subject | Email subject line |
| sender | Sender email address |
| urgency_score | AI-assigned urgency score (0-100) |
| urgency_reason | AI reasoning for the score |
| notified_at | When the Telegram alert was sent |
| feedback | User feedback: "useful" or "noise" |
| feedback_reason | User's explanation (extracted by LLM) |

### Cron Schedule

Runs as a post-hook on the Email Refresh cron job (every 30 min, 7am-8pm in `${TIMEZONE}`). The Node script handles time-window gating internally — cron fires during working hours and the script decides whether to send alerts based on the specific time window rules.

| Job | Schedule |
|-----|----------|
| Urgent Email Check | Post-hook on Email Refresh (every 30 min, 7am-8pm) |

## Steps

### Phase 1: Database Setup

#### 1.1 Verify or Create the Notifications Table `[AUTO]`

Check that the `urgent_notifications` table exists in the CRM database. If not, run the migration to create it.

**Remote:**
```
cd ${WORKSPACE}/crm && sqlite3 data/contacts.db '.tables' | grep urgent
```

Expected: `urgent_notifications` appears in the output.

If the table is missing, run the migration:

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/migrate.js
```

Expected: Migration completes successfully, table now exists.

If this fails: Check that `sqlite3` is available and the database file is not locked. If `migrate.js` doesn't exist, create the table directly with SQL (see the Storage section above for the schema).

If already exists: Skip — table is already in place.

### Phase 2: Install Scripts

#### 2.1 Verify Urgent Email Scripts `[AUTO]`

Confirm the urgent email scripts are in place and accessible.

**Remote:**
```
ls -la ${WORKSPACE}/crm/scripts/urgent-email-check.js ${WORKSPACE}/crm/scripts/urgent-feedback.js
```

Expected: Both files exist with reasonable file sizes (> 0 bytes).

If this fails: The scripts may not have been deployed with the CRM. Check if they need to be copied from the skill package or created. See the CRM deployment for the full script list.

#### 2.2 Configure Email Refresh Post-Hook `[GUIDED]`

Chain the urgent email check to run after the Email Refresh cron job completes. This ensures urgent scanning happens on freshly-ingested emails.

First, check the current Email Refresh configuration:

**Remote:**
```
openclaw cron list | grep 'Email Refresh'
```

Expected: Email Refresh cron job exists with a schedule like `*/30 7-20 * * *`.

Then add the post-hook. To route per-email classification through a cheaper model than the agent default, pass `URGENT_EMAIL_MODEL` as an env var:

**Remote:**
```
openclaw cron edit --name 'Email Refresh' \
  --env URGENT_EMAIL_MODEL=openai-codex/gpt-5.4-nano \
  --post-hook 'cd ${WORKSPACE}/crm && node scripts/urgent-email-check.js'
```

Expected: Cron job updated with the post-hook.

If this fails: If `openclaw cron edit` doesn't support `--post-hook`, edit the Email Refresh cron wrapper script directly to append the urgent check command at the end.

**Env var consumption in the node script.** `scripts/urgent-email-check.js` should read `URGENT_EMAIL_MODEL` and pass it through to whatever LLM client it uses. Reference pattern:

```javascript
// In scripts/urgent-email-check.js
const MODEL = process.env.URGENT_EMAIL_MODEL || '';  // empty → agent default
// If invoking openclaw agent:
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

If already exists: Check if the post-hook is already configured. If so, verify it points to the correct script path and skip.

### Phase 3: Configure Alerts

#### 3.1 Set Telegram CRM Topic `[GUIDED]`

Ensure urgent alerts route to the CRM topic in Telegram.

**Remote:**
```
openclaw config get crm.telegram-topic
```

Expected: Returns `${CRM_TOPIC_ID}`.

If not set or different:

**Remote:**
```
openclaw config set crm.telegram-topic ${CRM_TOPIC_ID}
```

Expected: Config updated confirmation.

If already exists: If the value matches `${CRM_TOPIC_ID}`, skip.

### Phase 4: Test and Verify

#### 4.1 Run a Manual Scan `[GUIDED]`

Run the urgent email scanner manually to verify it works.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/urgent-email-check.js
```

Expected: Script runs and scans recent emails. If running outside the alert time window, the script will scan but suppress notifications.

To bypass time gating for testing:

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/urgent-email-check.js --force
```

Expected: Any urgent emails trigger a Telegram alert in the CRM topic. If no emails are urgent, the script exits cleanly with no alerts.

If this fails: Check script output for errors. Common issues: missing Telegram bot token, Gmail auth expired, missing environment variables.

#### 4.2 Test the Feedback Loop `[GUIDED]`

If alerts were generated, test the feedback mechanism.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/urgent-feedback.js --latest
```

Expected: Shows the most recent notification and prompts for feedback (useful/noise).

#### 4.3 Verify Notification Storage `[AUTO]`

Check that notifications are being stored in the database.

**Remote:**
```
cd ${WORKSPACE}/crm && sqlite3 data/contacts.db 'SELECT id, subject, urgency_score, feedback FROM urgent_notifications ORDER BY notified_at DESC LIMIT 10;'
```

Expected: Recent notifications listed with urgency scores. Feedback column may be null for new entries.

## Verification

Run these checks to confirm the urgent email system is operational:

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/urgent-email-check.js --force
```

Check Telegram CRM topic for alerts.

**Remote:**
```
openclaw cron list | grep 'Email Refresh'
```

Expected:
- Email Refresh cron job has the urgent check post-hook
- Recent emails scanned without errors
- High-urgency emails trigger Telegram alerts
- Notifications stored in the `urgent_notifications` table

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No alerts generated | Outside time window or no urgent emails | Check time window rules. Use `--force` to bypass for testing. |
| Too many alerts (noise) | Classifier too sensitive | Give "wasn't urgent" feedback via `urgent-feedback.js`. Check the noise sender pre-filter list. |
| False negatives (missed urgent) | Classifier too conservative | Give "was urgent" feedback to improve recall. Check if sender was incorrectly pre-filtered. |
| Pre-filter too aggressive | Legitimate senders in noise list | Review noise sender list in script config. Remove incorrectly classified senders. |
| Duplicate notifications | Dedup not working | Check `email_id` column for duplicates. Script uses Gmail message ID as dedup key. |
| Feedback not improving | Too few samples | Few-shot learner needs several examples. Continue providing feedback consistently. |
| Email Refresh not triggering urgent check | Post-hook not configured | Verify Email Refresh cron has the post-hook. Re-run Step 2.2. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Verify/create table | Execute silently | Execute silently | Show output, confirm |
| 2.1 Verify scripts | Execute silently | Execute silently | Show file listing |
| 2.2 Configure post-hook | Execute silently | Confirm before modifying cron | Confirm command |
| 3.1 Set Telegram topic | Execute silently | Confirm topic ID | Confirm command |
| 4.1 Manual scan | Execute, report result | Execute, review alerts together | Confirm, review together |
| 4.2 Test feedback | Skip | Execute, show result | Execute, show result |
| 4.3 Verify storage | Execute silently | Execute, show entries | Show entries |

## Dependencies

- **Requires:** `deploy-personal-crm.md`, `deploy-google-workspace.md`, `deploy-messaging-setup.md`
- **Enhanced by:** Feedback over time (the more corrections, the better the classifier)
- **Required by:** None (standalone alert system)
