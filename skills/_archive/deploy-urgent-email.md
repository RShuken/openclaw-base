# Deploy Urgent Email Detection

## Purpose

Install an AI-powered email scanner that detects urgent emails and alerts via Telegram. Includes a feedback learning loop -- learns from "was/wasn't urgent" corrections to improve over time.

**When to use:** When the user wants proactive alerts for important emails without constant inbox monitoring.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null"
cmd --session <id> --command "gog gmail messages list --max-results 1 2>/dev/null"
```
- Is the CRM database deployed? (Required for notification tracking)
- Is Gmail access working?
- What are the client's waking hours? (Default time gates may not fit their schedule)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Email source | Gmail via `gog` CLI | Microsoft 365, IMAP | Client doesn't use Gmail |
| Alert time window | Weekdays 5-9pm, weekends 7am-9pm PST | Custom hours per client schedule | Client has different waking hours or timezone |
| Scan frequency | Every 30 min during waking hours | More/less frequent | Client preference and email volume |
| Alert channel | Telegram CRM topic | Discord, Slack, SMS | Client's preferred urgent notification channel |
| Noise filter | Pre-filter known noise senders | Custom sender blocklist | Client has different noise patterns |
| Feedback mechanism | "Was/wasn't urgent" replies in Telegram | CLI feedback, different channel | Client's interaction preference |

## Prerequisites

- `deploy-personal-crm.md` completed (CRM database for notification tracking)
- `deploy-google-workspace.md` completed (Gmail access)
- `deploy-messaging-setup.md` completed (Telegram CRM topic)

## What Gets Installed

### Scripts

| Script | Purpose |
|--------|---------|
| urgent-email-check.js | Scan recently-ingested inbound emails for urgency |
| urgent-feedback.js | Process feedback on notifications (useful/noise) |

### Features

- AI classification with few-shot learning from feedback
- Time-gated alerts: weekdays 5-9pm PST, weekends 7am-9pm PST (no middle-of-night alerts)
- Pre-filters noise senders (newsletters, marketing, automated)
- Dedicated Telegram topic for urgent-only alerts
- Feedback loop: reply "was/wasn't urgent" -> LLM extracts verdict and reasoning -> improves future classification
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

Runs after Email Refresh (every 30 min, 7am-8pm PST). The Node script handles time-window gating internally, so cron fires during working hours and the script decides whether to send alerts based on the specific time window rules.

| Job | Schedule |
|-----|----------|
| Urgent Email Check | After Email Refresh (every 30 min, 7am-8pm PST) |

## Steps

### 1. Verify CRM Database Has Notifications Table

Check that the `urgent_notifications` table exists. If not, the migration will create it.

```
cmd --session <id> --command "cd ~/clawd/crm && sqlite3 data/contacts.db '.tables' | grep urgent"
```

If the table is missing:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/migrate.js"
```

### 2. Install Urgent Email Scripts

Verify the scripts are in place.

```
cmd --session <id> --command "ls -la ~/clawd/crm/scripts/urgent-email-check.js ~/clawd/crm/scripts/urgent-feedback.js"
```

### 3. Configure Cron Wrapper

Install the shell wrapper that chains the urgent email check after the Email Refresh cron job.

```
cmd --session <id> --command "cat ~/clawd/crm/scripts/urgent-email-check.sh"
```

The wrapper should call `urgent-email-check.js` after the Email Refresh completes. If the wrapper needs updating:

```
cmd --session <id> --command "openclaw cron update --name 'Email Refresh' --post-hook 'cd ~/clawd/crm && node scripts/urgent-email-check.js'"
```

### 4. Configure Telegram CRM Topic

Ensure urgent alerts route to the CRM topic (ID: 709).

```
cmd --session <id> --command "openclaw config get crm.telegram-topic"
```

Should return `709`. If not set:

```
cmd --session <id> --command "openclaw config set crm.telegram-topic 709"
```

### 5. Test with a Manual Run

Run the urgent email scanner manually to verify it works.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/urgent-email-check.js"
```

Check Telegram for any urgent alerts. Note: if running outside the time window (weekdays 5-9pm, weekends 7am-9pm PST), the script will scan but suppress notifications. Use `--force` to bypass time gating for testing:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/urgent-email-check.js --force"
```

## Verification

Run these commands to confirm the urgent email system is operational:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/urgent-email-check.js"
```

Check Telegram for any urgent alerts. If alerts appeared, test the feedback loop:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/urgent-feedback.js --latest"
```

To see all tracked notifications:

```
cmd --session <id> --command "cd ~/clawd/crm && sqlite3 data/contacts.db 'SELECT id, subject, urgency_score, feedback FROM urgent_notifications ORDER BY notified_at DESC LIMIT 10;'"
```

Expected output:
- Recent emails scanned without errors
- High-urgency emails trigger Telegram alerts
- Feedback entries are stored and affect future classification

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No alerts generated | Outside time window or no urgent emails | Check the time window rules (weekdays 5-9pm, weekends 7am-9pm PST). Use `--force` to bypass for testing. |
| Too many alerts (noise) | Classifier too sensitive | Give "wasn't urgent" feedback via `urgent-feedback.js`. The few-shot learner will adjust. Check the noise sender pre-filter list. |
| False negatives (missed urgent emails) | Classifier too conservative | Give "was urgent" feedback to improve recall. Check if sender was incorrectly pre-filtered as noise. |
| Pre-filter too aggressive | Legitimate senders in the noise list | Review the noise sender list in the script config. Remove any incorrectly classified senders. |
| Duplicate notifications | Dedup not working | Check `email_id` column for duplicates. The script uses Gmail message ID as the dedup key. |
| Feedback not improving classification | Too few feedback samples | The few-shot learner needs several examples. Continue providing feedback consistently. |
| Email Refresh not triggering urgent check | Post-hook not configured | Verify the Email Refresh cron job has the post-hook set. Re-run Step 3. |

## Dependencies

- **Requires:** `deploy-personal-crm.md`, `deploy-google-workspace.md`, `deploy-messaging-setup.md`
