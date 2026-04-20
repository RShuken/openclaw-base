# Deploy Personal CRM

## Purpose

Install a personal CRM that automatically discovers contacts from Gmail and Google Calendar, tracks relationships with vector-based semantic search, and provides natural language queries via Telegram. The centerpiece data system -- feeds into briefings, meeting pipeline, email detection, and advisory council.

**When to use:** After Google Workspace is connected. This is typically the first major data system deployed.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null"
cmd --session <id> --command "gog auth status 2>/dev/null"
cmd --session <id> --command "echo $GEMINI_API_KEY | head -c 8"
```
- Is there an existing CRM database? If so, this is an upgrade, not a fresh install.
- Is Google Workspace connected? (Required for default email/calendar scanning)
- Is the embedding API key configured? (Required for vector search)
- Does the client want Box document linking? (Optional, most clients skip this)

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Email source | Gmail via `gog` CLI | Microsoft 365, IMAP, or manual-only | Client doesn't use Gmail |
| Calendar source | Google Calendar via `gog` | Outlook, CalDAV, or manual-only | Client doesn't use Google Calendar |
| Embedding provider | Google `gemini-embedding-001` (768-dim) | OpenAI `text-embedding-3-small` (1536-dim) | Client prefers OpenAI or doesn't have Gemini access |
| Document linking | Box integration (disabled by default) | Dropbox, SharePoint, local files, or none | Client uses different document storage |
| Task manager | Todoist (for Fathom action items) | Linear, Asana, none | Client uses different task management |
| Database path | `${WORKSPACE}/crm/data/contacts.db` | Custom path | Non-standard workspace |
| Telegram CRM topic | Topic ID 709 | Different ID, or Discord/Slack channel | Client's messaging setup |
| Sync schedule | 2am PST daily, email refresh every 30min | Adjust timezone and frequency | Client's timezone and preferences |
| Learning system | Auto-learns from approve/reject | Manual curation only | Client wants full control over contact additions |

## Prerequisites

- `deploy-google-workspace.md` completed (Gmail + Calendar access via gog)
- `deploy-messaging-setup.md` completed (Telegram CRM topic)
- `deploy-security-safety.md` completed (content sanitizer for email ingestion)
- Node.js + npm available
- SQLite3 available

## What Gets Installed

### SQLite Database (`~/clawd/crm/data/contacts.db`)

20 tables:

| Table | Purpose |
|-------|---------|
| contacts | Core contact info (name, email, company, role, priority, relationship_score) |
| interactions | Meeting/email/call/message log with fathom_meeting_id FK |
| follow_ups | Scheduled reminders with due dates, snoozing, status |
| contact_context | Timeline entries with 768-dim vector embeddings, direction, response time, topic tags |
| contact_summaries | LLM-generated relationship summaries with embeddings |
| meetings | Fathom meeting data (title, summary, transcript, attendees, action items) |
| meeting_action_items | Action items with assignee, ownership, status, due date, Todoist link |
| merge_suggestions | Duplicate detection with score, reasons, accept/decline workflow |
| relationship_profiles | Relationship analysis (type, style, topics, sentiment, interaction counts) |
| learning_patterns | Learned filtering patterns from approve/reject decisions |
| rejected_contacts | Tracking rejected candidates with rejection counts |
| sources | Contact discovery source tracking |
| meta | Key-value store for sync cursors, timestamps |
| box_files | Box file metadata |
| box_file_chunks | Text chunks with embeddings for semantic search |
| box_file_collaborators | Collaborator emails and roles per file |
| contact_document_links | Relevance links between contacts and Box files |
| box_sync_state | Box sync checkpoints |
| email_draft_requests | Gmail draft proposals with approval workflow |
| urgent_notifications | Urgent email tracking with feedback loop |

### Scripts

| Script | Purpose |
|--------|---------|
| sync.js | Manual contact discovery scan |
| daily-sync.js | Automated daily sync (cron at 2am PST) |
| batch-scan.js | Batch scanning with rate limiting, classification, auto-approval |
| query.js | CLI for natural language queries |
| merge-contacts.js | Merge duplicate contacts |
| health-check.js | Database health monitor (contact count drops, corruption) |

### Handler Modules

- contacts.js -- Contact lookup, company queries, merge, sync, stats
- follow-ups.js -- Create, list, mark done, snooze
- interactions.js -- Log from natural language, nudge generation
- topics.js -- Semantic topic search across contacts
- documents.js -- Box document relevance per contact

### Intelligence Modules

- Relationship scorer (0-100 health scores)
- Nudge generator (who needs attention)
- Pattern learner (improves from approve/reject decisions)
- Contact classifier and filter
- Duplicate detection with merge suggestions

### Natural Language Intent Detection (16 intents)

contact, topic, log_interaction, create_follow_up, list_follow_ups, mark_follow_up_done, snooze_follow_up, nudges, contact_documents, show_source, merge_suggestions, merge_accept/decline, merge, company, sync, stats

### Cron Jobs

| Job | Schedule | What |
|-----|----------|------|
| Daily CRM Ingestion | 2am PST | batch-scan.js with 1-day lookback |
| Email Refresh | Every 30min, 7am-8pm PST | Lightweight context refresh |

### Telegram Integration

- CRM topic (ID: 709) for queries, nudges, follow-ups
- Email topic (ID: 2229) for draft proposals

### Vector Embeddings

Google gemini-embedding-001 (768-dim) for semantic search across contact context and summaries.

## Steps

### 1. Create CRM Directory Structure

```
cmd --session <id> --command "mkdir -p ~/clawd/crm/data ~/clawd/crm/src ~/clawd/crm/scripts"
```

### 2. Install Dependencies

```
cmd --session <id> --command "cd ~/clawd/crm && npm install"
```

### 3. Initialize Database

Run the migration script to create all 20 tables with proper indexes and WAL mode.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/migrate.js"
```

### 4. Run Initial Contact Discovery

Scan Gmail and Calendar for contacts. This may take several minutes depending on email history.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/sync.js"
```

### 5. Review Discovered Contacts

Batch scan classifies contacts and presents them for approve/reject. The pattern learner improves over time.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/batch-scan.js"
```

### 6. Set Up Daily Sync Cron (2am PST)

Add the daily ingestion job to the cron scheduler.

```
cmd --session <id> --command "openclaw cron add --name 'Daily CRM Ingestion' --schedule '0 2 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/crm && node scripts/daily-sync.js'"
```

### 7. Set Up Email Refresh Cron (Every 30min, 7am-8pm PST)

Lightweight refresh of email context during working hours.

```
cmd --session <id> --command "openclaw cron add --name 'Email Refresh' --schedule '*/30 7-20 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/crm && node scripts/sync.js --email-only --lightweight'"
```

### 8. Install CRM Query Skill

Register the crm-query skill so the agent can handle natural language CRM queries.

```
cmd --session <id> --command "openclaw skill install crm-query --handler ~/clawd/crm/src/contacts.js"
```

### 9. Configure Telegram Topic for CRM Queries

Route CRM messages to topic 709 and email drafts to topic 2229.

```
cmd --session <id> --command "openclaw config set crm.telegram-topic 709"
cmd --session <id> --command "openclaw config set crm.email-topic 2229"
```

### 10. Verify Natural Language Queries

Test that the query pipeline is working end-to-end.

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/query.js 'How many contacts?'"
```

## Verification

Run these commands to confirm the CRM is fully operational:

```
cmd --session <id> --command "cd ~/clawd/crm && node scripts/query.js 'stats'"
cmd --session <id> --command "cd ~/clawd/crm && node scripts/query.js 'Who do I know at Google?'"
cmd --session <id> --command "cd ~/clawd/crm && node scripts/health-check.js"
```

Expected output:
- Stats should show contact count, interaction count, follow-up count
- Company query should return matching contacts with relationship scores
- Health check should report all tables present, no corruption, no count anomalies

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No contacts found | gog auth expired or Gmail has no email history | Re-run `gog auth login` and verify Gmail scopes include read access. |
| Embedding errors | GEMINI_API_KEY not set or invalid | Verify `GEMINI_API_KEY` is set in `~/.openclaw/.env`. Test with `curl` against the embedding endpoint. |
| Slow sync | Too many concurrent API calls | Reduce concurrency in batch-scan.js (default: 3). Use `--concurrency 1` flag. |
| Database locked | Stale process holding WAL lock | Check for zombie processes: `lsof ~/clawd/crm/data/contacts.db`. Kill stale processes. SQLite uses WAL mode so reads should not block. |
| Duplicate contacts appearing | Merge detection not run | Run `node scripts/merge-contacts.js --suggest` to generate merge suggestions, then review in Telegram. |
| Query returns wrong intent | Intent classifier confused | Check the 16 intent patterns. Add explicit keywords to the query for disambiguation. |

## Dependencies

- **Requires:** `deploy-google-workspace.md`, `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Required by:** `deploy-fathom-pipeline.md`, `deploy-urgent-email.md`, `deploy-daily-briefing.md`, `deploy-advisory-council.md`
