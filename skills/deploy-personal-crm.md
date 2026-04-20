# Deploy Personal CRM

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-19 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale. On 2026.4.15: `openclaw cron`, `openclaw config` (get/set/unset/file/validate), `openclaw skills` (list/install/check/info/search/update) all exist per `audit/_baseline.md` §3/§4C.4/§4C.7. The skill name was wrong (`skill` → `skills`).
- **Scheduling**: `openclaw cron add --name crm-discover --cron "0 2 * * *"` replaces every "platform-native scheduling" reference
- **Config**: `openclaw config set` for scalars; for the 20-table CRM schema edit `~/.openclaw/openclaw.json` / `.env` directly
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`CRM_INTENT_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to natural-language intent detection for Telegram queries.
- **ClawHub alternatives**: `personal-crm`, `heleni-personal-crm`, `crm` — evaluate per `audit/_baseline.md` §B before scratch-building

## Purpose

Install a personal CRM that automatically discovers contacts from Gmail and Google Calendar, tracks relationships with vector-based semantic search, and provides natural language queries via Telegram. This is the centerpiece data system -- it feeds into daily briefings, the Fathom meeting pipeline, urgent email detection, and the advisory council.

**When to use:** After Google Workspace is connected. This is typically the first major data system deployed.

**What this skill does:**
1. Creates the CRM directory structure and initializes the 20-table SQLite database with full schema
2. Installs Node.js dependencies for contact discovery, embedding, and query modules
3. Runs initial contact discovery from Gmail and Google Calendar
4. Configures the batch scan classifier and presents contacts for approve/reject
5. Sets up cron jobs for daily ingestion and working-hours email refresh
6. Registers the CRM query skill and Telegram topic routing
7. Verifies end-to-end natural language query pipeline

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${GEMINI_API_KEY}` | Client profile: API keys, or `~/.openclaw/.env` | `AIzaSy...` |
| `${CRM_TOPIC_ID}` | Created during messaging setup (Telegram CRM topic) | `709` |
| `${EMAIL_TOPIC_ID}` | Created during messaging setup (Telegram email topic) | `2229` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${SYNC_HOUR}` | Client preference: daily sync hour (24h format) | `2` |
| `${WORK_START_HOUR}` | Client preference: working hours start | `7` |
| `${WORK_END_HOUR}` | Client preference: working hours end | `20` |
| `${EMBEDDING_PROVIDER}` | Client profile: embedding preference | `gemini` or `openai` |
| `${EMBEDDING_MODEL}` | Derived from provider | `gemini-embedding-001` or `text-embedding-3-small` |
| `${EMBEDDING_DIM}` | Derived from model | `768` (Gemini) or `1536` (OpenAI) |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the CRM database already exists (this determines fresh install vs. upgrade):

**Remote:**
```
ls ${WORKSPACE}/crm/data/contacts.db 2>/dev/null && echo 'EXISTS' || echo 'NOT_FOUND'
```

Verify Google Workspace is connected:

**Remote:**
```
gog auth status 2>/dev/null && echo 'GOG_OK' || echo 'GOG_MISSING'
```

Verify the embedding API key is configured:

**Remote:**
```
echo $GEMINI_API_KEY | head -c 8
```

Expected: First 8 characters of the API key, or empty if not set.

Check SQLite availability:

**Remote:**
```
sqlite3 --version 2>/dev/null || echo 'NO_SQLITE'
```

**Decision points from pre-flight:**
- Is there an existing CRM database? If so, this is an upgrade -- back up before proceeding and run migrations only.
- Is Google Workspace connected? Required for default email/calendar scanning. If not, deploy `deploy-google-workspace.md` first.
- Is the embedding API key configured? Required for vector search. If missing, set it in `~/.openclaw/.env`.
- Does the client want Box document linking? Optional -- most clients skip this. The schema includes Box tables but they remain empty unless configured.

## Adaptation Points

The defaults below work for most installs. Cross-reference against the client profile for overrides.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Email source | Gmail via `gog` CLI | Client doesn't use Gmail (Microsoft 365, IMAP, or manual-only) |
| Calendar source | Google Calendar via `gog` | Client doesn't use Google Calendar (Outlook, CalDAV, or manual-only) |
| Embedding provider | Google `gemini-embedding-001` (768-dim) | Client prefers OpenAI or doesn't have Gemini access; switch to `text-embedding-3-small` (1536-dim) |
| Document linking | Box integration (disabled by default) | Client uses Box, Dropbox, SharePoint, or local files |
| Task manager | Todoist (for Fathom action items) | Client uses Linear, Asana, or none |
| Database path | `${WORKSPACE}/crm/data/contacts.db` | Non-standard workspace layout |
| Telegram CRM topic | `${CRM_TOPIC_ID}` | Client uses Discord or Slack instead of Telegram |
| Sync schedule | Daily at `${SYNC_HOUR}` in `${TIMEZONE}`, email refresh every 30min during working hours | Adjust timezone, frequency, or working hours |
| Learning system | Auto-learns from approve/reject decisions | Client wants manual curation only |

## Prerequisites

- `openclaw/install.md` completed (OpenClaw installed and running)
- `deploy-google-workspace.md` completed (Gmail + Calendar access via gog)
- `deploy-messaging-setup.md` completed (Telegram topics created, including CRM topic)
- `deploy-security-safety.md` completed (content sanitizer for email ingestion)
- Node.js + npm available on the client machine
- SQLite3 available on the client machine

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| CRM database | `${WORKSPACE}/crm/data/contacts.db` | 20-table SQLite database with WAL mode |
| Source modules | `${WORKSPACE}/crm/src/` | Handler modules (contacts, follow-ups, interactions, topics, documents) |
| Scripts | `${WORKSPACE}/crm/scripts/` | Sync, batch-scan, query, merge, health-check |
| npm packages | `${WORKSPACE}/crm/node_modules/` | better-sqlite3, node-fetch, etc. |
| CRM query skill | Registered via `openclaw skills install` | Natural language intent detection (16 intents) |
| Daily sync cron | `openclaw cron` | 2am daily batch ingestion |
| Email refresh cron | `openclaw cron` | Every 30min during working hours |

### Natural Language Intent Detection (16 intents)

contact, topic, log_interaction, create_follow_up, list_follow_ups, mark_follow_up_done, snooze_follow_up, nudges, contact_documents, show_source, merge_suggestions, merge_accept, merge_decline, merge, company, sync, stats

## Steps

### Phase 1: Create Directory Structure and Database

#### 1.1 Create CRM directories `[AUTO]`

Create the CRM directory tree for data, source modules, and scripts.

**Remote:**
```
mkdir -p ${WORKSPACE}/crm/data ${WORKSPACE}/crm/src ${WORKSPACE}/crm/scripts
```

Expected: Three directories created. No output on success.

If this fails: Check that `${WORKSPACE}` exists. If not, create it first with `mkdir -p ${WORKSPACE}`.

If already exists: `mkdir -p` is idempotent -- safe to re-run.

#### 1.2 Initialize the CRM database schema `[GUIDED]`

Create the SQLite database with all 20 tables, indexes, and WAL mode. This is the core data model -- **use the exact SQL below** (Level 1: exact syntax).

**Remote:**
```
sqlite3 ${WORKSPACE}/crm/data/contacts.db << 'SCHEMA_EOF'
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;

-- Core contact info
CREATE TABLE IF NOT EXISTS contacts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE,
  phone TEXT,
  company TEXT,
  role TEXT,
  title TEXT,
  linkedin_url TEXT,
  twitter_handle TEXT,
  notes TEXT,
  priority INTEGER DEFAULT 0,
  relationship_score REAL DEFAULT 0.0,
  status TEXT DEFAULT 'active' CHECK(status IN ('active','archived','merged')),
  source TEXT,
  discovered_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  last_interaction_at TEXT,
  merged_into_id INTEGER REFERENCES contacts(id)
);

-- Meeting/email/call/message log
CREATE TABLE IF NOT EXISTS interactions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL REFERENCES contacts(id),
  type TEXT NOT NULL CHECK(type IN ('email','meeting','call','message','note')),
  direction TEXT CHECK(direction IN ('inbound','outbound','mutual')),
  subject TEXT,
  summary TEXT,
  raw_content TEXT,
  sentiment REAL,
  fathom_meeting_id TEXT,
  occurred_at TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Scheduled reminders with due dates
CREATE TABLE IF NOT EXISTS follow_ups (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL REFERENCES contacts(id),
  description TEXT NOT NULL,
  due_date TEXT NOT NULL,
  status TEXT DEFAULT 'pending' CHECK(status IN ('pending','done','snoozed','cancelled')),
  snoozed_until TEXT,
  priority INTEGER DEFAULT 0,
  created_at TEXT DEFAULT (datetime('now')),
  completed_at TEXT
);

-- Timeline entries with vector embeddings
CREATE TABLE IF NOT EXISTS contact_context (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL REFERENCES contacts(id),
  content TEXT NOT NULL,
  content_type TEXT NOT NULL,
  direction TEXT CHECK(direction IN ('inbound','outbound','mutual')),
  response_time_hours REAL,
  topic_tags TEXT,
  embedding BLOB,
  occurred_at TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);

-- LLM-generated relationship summaries
CREATE TABLE IF NOT EXISTS contact_summaries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL REFERENCES contacts(id),
  summary TEXT NOT NULL,
  summary_type TEXT DEFAULT 'relationship',
  embedding BLOB,
  generated_at TEXT DEFAULT (datetime('now')),
  valid_until TEXT
);

-- Fathom meeting data
CREATE TABLE IF NOT EXISTS meetings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  fathom_meeting_id TEXT UNIQUE,
  title TEXT,
  summary TEXT,
  transcript TEXT,
  attendees TEXT,
  action_items TEXT,
  occurred_at TEXT NOT NULL,
  duration_minutes INTEGER,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Action items with assignee and status
CREATE TABLE IF NOT EXISTS meeting_action_items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  meeting_id INTEGER REFERENCES meetings(id),
  contact_id INTEGER REFERENCES contacts(id),
  description TEXT NOT NULL,
  assignee TEXT,
  ownership TEXT CHECK(ownership IN ('mine','theirs','shared')),
  status TEXT DEFAULT 'open' CHECK(status IN ('open','done','cancelled')),
  due_date TEXT,
  todoist_task_id TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  completed_at TEXT
);

-- Duplicate detection with merge workflow
CREATE TABLE IF NOT EXISTS merge_suggestions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id_a INTEGER NOT NULL REFERENCES contacts(id),
  contact_id_b INTEGER NOT NULL REFERENCES contacts(id),
  score REAL NOT NULL,
  reasons TEXT,
  status TEXT DEFAULT 'pending' CHECK(status IN ('pending','accepted','declined')),
  created_at TEXT DEFAULT (datetime('now')),
  resolved_at TEXT
);

-- Relationship analysis profiles
CREATE TABLE IF NOT EXISTS relationship_profiles (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL UNIQUE REFERENCES contacts(id),
  relationship_type TEXT,
  communication_style TEXT,
  preferred_topics TEXT,
  sentiment_trend REAL,
  total_interactions INTEGER DEFAULT 0,
  inbound_count INTEGER DEFAULT 0,
  outbound_count INTEGER DEFAULT 0,
  avg_response_time_hours REAL,
  last_analyzed_at TEXT,
  updated_at TEXT DEFAULT (datetime('now'))
);

-- Learned filtering patterns from approve/reject
CREATE TABLE IF NOT EXISTS learning_patterns (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  pattern_type TEXT NOT NULL,
  pattern_value TEXT NOT NULL,
  action TEXT NOT NULL CHECK(action IN ('approve','reject')),
  confidence REAL DEFAULT 0.5,
  occurrences INTEGER DEFAULT 1,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);

-- Rejected contact candidates
CREATE TABLE IF NOT EXISTS rejected_contacts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT NOT NULL,
  name TEXT,
  source TEXT,
  rejection_reason TEXT,
  rejection_count INTEGER DEFAULT 1,
  first_rejected_at TEXT DEFAULT (datetime('now')),
  last_rejected_at TEXT DEFAULT (datetime('now'))
);

-- Contact discovery source tracking
CREATE TABLE IF NOT EXISTS sources (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL REFERENCES contacts(id),
  source_type TEXT NOT NULL,
  source_id TEXT,
  discovered_at TEXT DEFAULT (datetime('now'))
);

-- Key-value store for sync cursors
CREATE TABLE IF NOT EXISTS meta (
  key TEXT PRIMARY KEY,
  value TEXT,
  updated_at TEXT DEFAULT (datetime('now'))
);

-- Box file metadata
CREATE TABLE IF NOT EXISTS box_files (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  box_file_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  extension TEXT,
  size_bytes INTEGER,
  parent_folder_id TEXT,
  parent_folder_name TEXT,
  sha1 TEXT,
  content_modified_at TEXT,
  synced_at TEXT DEFAULT (datetime('now'))
);

-- Text chunks with embeddings for Box semantic search
CREATE TABLE IF NOT EXISTS box_file_chunks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  box_file_id TEXT NOT NULL REFERENCES box_files(box_file_id),
  chunk_index INTEGER NOT NULL,
  content TEXT NOT NULL,
  embedding BLOB,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Collaborator emails and roles per Box file
CREATE TABLE IF NOT EXISTS box_file_collaborators (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  box_file_id TEXT NOT NULL REFERENCES box_files(box_file_id),
  email TEXT NOT NULL,
  role TEXT,
  name TEXT
);

-- Relevance links between contacts and Box files
CREATE TABLE IF NOT EXISTS contact_document_links (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER NOT NULL REFERENCES contacts(id),
  box_file_id TEXT NOT NULL REFERENCES box_files(box_file_id),
  relevance_score REAL,
  link_reason TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Box sync checkpoints
CREATE TABLE IF NOT EXISTS box_sync_state (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  folder_id TEXT NOT NULL,
  last_sync_at TEXT,
  sync_cursor TEXT,
  status TEXT DEFAULT 'active'
);

-- Gmail draft proposals with approval workflow
CREATE TABLE IF NOT EXISTS email_draft_requests (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  contact_id INTEGER REFERENCES contacts(id),
  to_email TEXT NOT NULL,
  subject TEXT NOT NULL,
  body TEXT NOT NULL,
  context TEXT,
  status TEXT DEFAULT 'pending' CHECK(status IN ('pending','approved','rejected','sent')),
  gmail_draft_id TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  resolved_at TEXT
);

-- Urgent email tracking with feedback loop
CREATE TABLE IF NOT EXISTS urgent_notifications (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email_id TEXT NOT NULL,
  contact_id INTEGER REFERENCES contacts(id),
  subject TEXT,
  urgency_score REAL,
  urgency_reason TEXT,
  feedback TEXT CHECK(feedback IN ('correct','false_positive','missed')),
  notified_at TEXT DEFAULT (datetime('now')),
  feedback_at TEXT
);

-- Indexes for performance
CREATE INDEX IF NOT EXISTS idx_contacts_email ON contacts(email);
CREATE INDEX IF NOT EXISTS idx_contacts_company ON contacts(company);
CREATE INDEX IF NOT EXISTS idx_contacts_status ON contacts(status);
CREATE INDEX IF NOT EXISTS idx_contacts_relationship_score ON contacts(relationship_score DESC);
CREATE INDEX IF NOT EXISTS idx_interactions_contact_id ON interactions(contact_id);
CREATE INDEX IF NOT EXISTS idx_interactions_occurred_at ON interactions(occurred_at);
CREATE INDEX IF NOT EXISTS idx_interactions_fathom_meeting_id ON interactions(fathom_meeting_id);
CREATE INDEX IF NOT EXISTS idx_follow_ups_contact_id ON follow_ups(contact_id);
CREATE INDEX IF NOT EXISTS idx_follow_ups_status ON follow_ups(status);
CREATE INDEX IF NOT EXISTS idx_follow_ups_due_date ON follow_ups(due_date);
CREATE INDEX IF NOT EXISTS idx_contact_context_contact_id ON contact_context(contact_id);
CREATE INDEX IF NOT EXISTS idx_contact_summaries_contact_id ON contact_summaries(contact_id);
CREATE INDEX IF NOT EXISTS idx_meetings_fathom_meeting_id ON meetings(fathom_meeting_id);
CREATE INDEX IF NOT EXISTS idx_meeting_action_items_meeting_id ON meeting_action_items(meeting_id);
CREATE INDEX IF NOT EXISTS idx_meeting_action_items_contact_id ON meeting_action_items(contact_id);
CREATE INDEX IF NOT EXISTS idx_merge_suggestions_status ON merge_suggestions(status);
CREATE INDEX IF NOT EXISTS idx_learning_patterns_type ON learning_patterns(pattern_type, pattern_value);
CREATE INDEX IF NOT EXISTS idx_rejected_contacts_email ON rejected_contacts(email);
CREATE INDEX IF NOT EXISTS idx_sources_contact_id ON sources(contact_id);
CREATE INDEX IF NOT EXISTS idx_box_files_box_file_id ON box_files(box_file_id);
CREATE INDEX IF NOT EXISTS idx_box_file_chunks_box_file_id ON box_file_chunks(box_file_id);
CREATE INDEX IF NOT EXISTS idx_box_file_collaborators_email ON box_file_collaborators(email);
CREATE INDEX IF NOT EXISTS idx_contact_document_links_contact_id ON contact_document_links(contact_id);
CREATE INDEX IF NOT EXISTS idx_email_draft_requests_status ON email_draft_requests(status);
CREATE INDEX IF NOT EXISTS idx_urgent_notifications_email_id ON urgent_notifications(email_id);
SCHEMA_EOF
```

Expected: No output (SQLite exits silently on success). Verify with:

**Remote:**
```
sqlite3 ${WORKSPACE}/crm/data/contacts.db "SELECT count(*) FROM sqlite_master WHERE type='table';"
```

Expected: `20` (twenty tables).

If this fails: Check that SQLite3 is installed (`sqlite3 --version`). If the command exceeds the 8192-byte API limit, split the schema into two commands -- tables first, then indexes.

If already exists: All statements use `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS` -- safe to re-run. Existing data is preserved.

> **Note on embedding dimensions:** The schema uses BLOB columns for embeddings. The actual dimension (768 for Gemini, 1536 for OpenAI) is handled by the application code, not the schema. No schema change is needed when switching embedding providers.

### Phase 2: Install Dependencies and Source Modules

#### 2.1 Initialize npm project and install dependencies `[AUTO]`

Set up the Node.js project with the required packages for SQLite, HTTP, and embedding support.

**Remote:**
```
cd ${WORKSPACE}/crm && npm init -y && npm install better-sqlite3 node-fetch
```

Expected: `package.json` created, `node_modules/` populated. No errors in npm output.

If this fails: Check Node.js and npm are available (`node --version && npm --version`). If `better-sqlite3` fails to compile, ensure build tools are installed (macOS: Xcode command line tools; Linux: `build-essential`).

If already exists: `npm install` is idempotent -- it skips packages already in `node_modules/`.

#### 2.2 Write handler modules `[GUIDED]`

Create the five handler modules in `${WORKSPACE}/crm/src/`. Each module handles a category of CRM operations.

The operator should create these files based on the intent detection system. Each handler exports functions that the CRM query skill dispatches to:

| Module | File | Handles |
|--------|------|---------|
| Contacts | `${WORKSPACE}/crm/src/contacts.js` | Contact lookup, company queries, merge, sync, stats |
| Follow-ups | `${WORKSPACE}/crm/src/follow-ups.js` | Create, list, mark done, snooze follow-ups |
| Interactions | `${WORKSPACE}/crm/src/interactions.js` | Log from natural language, nudge generation |
| Topics | `${WORKSPACE}/crm/src/topics.js` | Semantic topic search across contacts using embeddings |
| Documents | `${WORKSPACE}/crm/src/documents.js` | Box document relevance per contact (optional) |

Each module needs:
- Database connection to `${WORKSPACE}/crm/data/contacts.db` via `better-sqlite3`
- Embedding function using the configured provider (`${EMBEDDING_PROVIDER}`)
- Export of handler functions that accept a parsed intent and return a formatted response

> **Note:** These modules are substantial (hundreds of lines each). Write them to the client machine using the session commands API. For files exceeding ~5.5KB, use chunked base64 writes (see `e2e/lib/test-helpers.ts:writeFileCommands()` pattern).

Expected: Five `.js` files in `${WORKSPACE}/crm/src/`, each importable without errors.

If this fails: Check for syntax errors with `node -c ${WORKSPACE}/crm/src/contacts.js` (syntax check without executing).

#### 2.3 Write utility scripts `[GUIDED]`

Create the operational scripts in `${WORKSPACE}/crm/scripts/`:

| Script | Purpose | Key Behavior |
|--------|---------|-------------|
| `sync.js` | Manual contact discovery scan | Scans Gmail + Calendar via `gog`, classifies contacts, writes to DB |
| `daily-sync.js` | Automated daily sync | Same as sync.js but with 1-day lookback window, runs unattended |
| `batch-scan.js` | Batch scanning with rate limiting | Classification, auto-approval based on learning patterns, presents unknowns for review |
| `query.js` | CLI for natural language queries | Intent detection across 16 intents, dispatches to handler modules |
| `merge-contacts.js` | Merge duplicate contacts | Generates merge suggestions, accepts/declines, performs merge |
| `health-check.js` | Database health monitor | Checks table counts, detects drops, WAL integrity, index health |

**Remote (example for each script -- adapt content to the client's configuration):**
```
node -c ${WORKSPACE}/crm/scripts/sync.js && echo 'SYNTAX_OK'
```

Expected: `SYNTAX_OK` for each script after writing.

If this fails: Fix syntax errors reported by `node -c`. Common issues: missing `require()` paths, unclosed brackets.

### Phase 3: Initial Contact Discovery

#### 3.1 Run initial contact discovery `[HUMAN_INPUT]`

Scan Gmail and Google Calendar for contacts. This may take several minutes depending on email history. The client should be aware this will read their email metadata.

> **Ask the client:** "We're about to scan your Gmail and Calendar to discover contacts. This reads email headers and calendar attendees -- not email bodies. It may take several minutes. Ready to proceed?"

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/sync.js
```

Expected: Output showing number of contacts discovered, categorized by source (email, calendar). Typical first run discovers 50-500 contacts.

If this fails:
- `gog auth` errors: Re-authenticate with `gog auth login`. Ensure Gmail read scope is granted.
- Embedding API errors: Check `${GEMINI_API_KEY}` is set correctly. Test directly: `curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-embedding-001:embedContent?key=${GEMINI_API_KEY}" -H 'Content-Type: application/json' -d '{"model":"models/gemini-embedding-001","content":{"parts":[{"text":"test"}]}}'`
- Timeout: For large email histories, the first sync can take 5-10 minutes. Let it run.

#### 3.2 Run batch classification `[HUMAN_INPUT]`

Present discovered contacts for approve/reject review. The pattern learner improves classification accuracy over time.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/batch-scan.js
```

Expected: Contacts are classified and presented in batches. The operator relays approve/reject decisions. After several rounds, the learner auto-approves high-confidence matches.

If this fails: Check that contacts were discovered in step 3.1 (`sqlite3 ${WORKSPACE}/crm/data/contacts.db "SELECT count(*) FROM contacts;"`).

### Phase 4: Schedule Automated Jobs

#### 4.1 Add daily CRM ingestion cron `[AUTO]`

Schedule the nightly batch sync to run at the client's preferred time, scanning the previous day's emails and calendar events.

**Remote:**
```
openclaw cron add --name 'Daily CRM Ingestion' --schedule '0 ${SYNC_HOUR} * * *' --tz '${TIMEZONE}' \
  --env CRM_INTENT_MODEL=openai-codex/gpt-5.4-nano \
  --command 'cd ${WORKSPACE}/crm && node scripts/daily-sync.js'
```

`CRM_INTENT_MODEL` routes any LLM calls during ingestion (intent detection, contact classification) through a specific model. Omit to defer to the agent default. In `scripts/daily-sync.js` and the Telegram NL-query handler:

```javascript
const MODEL = process.env.CRM_INTENT_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

Expected: Cron job registered. Verify with `openclaw cron list`.

If this fails: Check `openclaw` is on PATH. Verify cron system is working with `openclaw cron list`.

If already exists: Check for duplicate entries with `openclaw cron list | grep 'CRM Ingestion'`. If found, update rather than add.

#### 4.2 Add email refresh cron `[AUTO]`

Schedule lightweight email context refresh during working hours. This keeps the CRM current without the full batch processing overhead.

**Remote:**
```
openclaw cron add --name 'Email Refresh' --schedule '*/${REFRESH_INTERVAL:-30} ${WORK_START_HOUR}-${WORK_END_HOUR} * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/crm && node scripts/sync.js --email-only --lightweight'
```

> **Note:** The operator must substitute the actual values for `${WORK_START_HOUR}`, `${WORK_END_HOUR}`, and interval. For example: `*/30 7-20 * * *` means every 30 minutes from 7am to 8pm.

Expected: Cron job registered. Verify with `openclaw cron list`.

If this fails: Same troubleshooting as step 4.1.

If already exists: Check for duplicates with `openclaw cron list | grep 'Email Refresh'`. Update if schedule changed.

### Phase 5: Register Skill and Configure Messaging

#### 5.1 Install CRM query skill `[GUIDED]`

Register the CRM query skill with OpenClaw so the agent can handle natural language CRM queries through Telegram and other channels.

**Remote:**
```
openclaw skills install crm-query --handler ${WORKSPACE}/crm/src/contacts.js
```

Expected: Skill registered. Verify with `openclaw skills list | grep crm-query`.

If this fails: Check the handler file exists and has no syntax errors. Verify with `node -c ${WORKSPACE}/crm/src/contacts.js`.

If already exists: Re-installing updates the handler path. Safe to re-run.

#### 5.2 Configure Telegram topic routing `[GUIDED]`

Route CRM messages to the designated Telegram topics. CRM queries go to the CRM topic; email draft proposals go to the email topic.

**Remote:**
```
openclaw config set crm.telegram-topic ${CRM_TOPIC_ID}
openclaw config set crm.email-topic ${EMAIL_TOPIC_ID}
```

Expected: Config values set. Verify with `openclaw config get crm.telegram-topic`.

If this fails: Check that OpenClaw config system is working (`openclaw config list`). If the topic IDs are unknown, check the client profile or run `deploy-messaging-setup.md` first.

If already exists: Setting config values is idempotent -- overwrites with the same value.

### Phase 6: Verify End-to-End

#### 6.1 Test natural language queries `[GUIDED]`

Verify the full query pipeline: intent detection -> handler dispatch -> database query -> formatted response.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/query.js 'How many contacts do I have?'
```

Expected: A count of contacts in the database.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/query.js 'Who do I know at Google?'
```

Expected: A list of contacts with company matching "Google", including relationship scores.

If this fails: Check each layer -- is the database populated? (`sqlite3 ${WORKSPACE}/crm/data/contacts.db "SELECT count(*) FROM contacts;"`). Does the handler load? (`node -e "require('${WORKSPACE}/crm/src/contacts.js')"`). Is the embedding API responding?

#### 6.2 Run health check `[AUTO]`

Verify database integrity, table counts, and index health.

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/health-check.js
```

Expected: All tables present (20), no corruption, no count anomalies, WAL mode active.

If this fails: Check for database corruption with `sqlite3 ${WORKSPACE}/crm/data/contacts.db "PRAGMA integrity_check;"`. Expected: `ok`.

#### 6.3 Update client profile `[AUTO]`

**Operator:** Update `clients/<name>.md` with:
- CRM deployed: date, version, contact count
- Database location: `${WORKSPACE}/crm/data/contacts.db`
- Embedding provider: `${EMBEDDING_PROVIDER}` / `${EMBEDDING_MODEL}`
- Telegram topics: CRM topic `${CRM_TOPIC_ID}`, email topic `${EMAIL_TOPIC_ID}`
- Cron jobs: daily ingestion at `${SYNC_HOUR}`, email refresh every 30min
- Any issues encountered or adaptations made

## Verification

Run these checks to confirm the CRM is fully operational.

**Database health:**

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/health-check.js
```

Expected: All 20 tables present, no corruption, WAL mode active.

**Contact stats:**

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/query.js 'stats'
```

Expected: Contact count, interaction count, follow-up count, and active/archived breakdown.

**Semantic search:**

**Remote:**
```
cd ${WORKSPACE}/crm && node scripts/query.js 'Who do I know at Google?'
```

Expected: Matching contacts with relationship scores. This confirms embedding search is working.

**Cron jobs:**

**Remote:**
```
openclaw cron list | grep -E 'CRM|Email'
```

Expected: Two cron entries -- "Daily CRM Ingestion" and "Email Refresh".

**Skill registration:**

**Remote:**
```
openclaw skills list | grep crm-query
```

Expected: `crm-query` skill listed as active.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No contacts found after sync | `gog` auth expired or Gmail has no history | Re-run `gog auth login`, verify Gmail scopes include read access |
| Embedding errors | `GEMINI_API_KEY` not set or invalid | Check `~/.openclaw/.env` for the key. Test with a direct `curl` to the embedding endpoint |
| Slow initial sync | Too many concurrent API calls | Reduce concurrency in batch-scan.js (default: 3). Use `--concurrency 1` flag |
| Database locked | Stale process holding WAL lock | Check for zombie processes: `lsof ${WORKSPACE}/crm/data/contacts.db`. Kill stale processes. WAL mode allows concurrent reads |
| Duplicate contacts appearing | Merge detection not run | Run `node scripts/merge-contacts.js --suggest` to generate merge suggestions, then review |
| Query returns wrong intent | Intent classifier confused | Check the 16 intent patterns. Add explicit keywords to the query for disambiguation |
| `better-sqlite3` won't install | Missing build tools | macOS: `xcode-select --install`. Linux: `sudo apt install build-essential`. Windows: install Windows Build Tools |
| Schema command exceeds 8KB limit | Full schema SQL too large for single API call | Split into two commands: tables first (up to `urgent_notifications`), then indexes in a second command |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Create directories | Execute silently | Execute silently | Confirm before creating |
| 1.2 Initialize database schema | Execute silently | Confirm before running SQL | Show schema, confirm |
| 2.1 Install npm dependencies | Execute silently | Confirm before install | Confirm |
| 2.2 Write handler modules | Execute silently | Confirm module list, then execute | Confirm each file |
| 2.3 Write utility scripts | Execute silently | Confirm script list, then execute | Confirm each file |
| 3.1 Initial contact discovery | Execute (always notify client) | Ask client, then execute | Ask client, relay progress |
| 3.2 Batch classification | Auto-approve high-confidence | Present batches to operator | Present individual contacts |
| 4.1 Daily sync cron | Execute silently | Confirm schedule | Confirm schedule and command |
| 4.2 Email refresh cron | Execute silently | Confirm schedule | Confirm schedule and command |
| 5.1 Install CRM skill | Execute silently | Confirm | Confirm |
| 5.2 Configure Telegram topics | Execute silently | Confirm topic IDs | Confirm each config value |
| 6.1 Test queries | Execute and report results | Execute and report results | Execute each query, discuss results |
| 6.2 Health check | Execute and report | Execute and report | Execute, review output |
| 6.3 Update client profile | Execute silently | Show changes, confirm | Show changes, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-google-workspace.md`, `deploy-messaging-setup.md`, `deploy-security-safety.md`
- **Required by:** `deploy-fathom-pipeline.md`, `deploy-urgent-email.md`, `deploy-daily-briefing.md`, `deploy-advisory-council.md`
- **Enhanced by:** `deploy-newsletter-crm.md` (adds HubSpot sync), `deploy-asana.md` (adds task integration)
