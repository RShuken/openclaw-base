# Deploy Action Item Extraction and Approval Engine

> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT.** This file was preserved from the {{PRINCIPAL_NAME}} / {{COMPANY_NAME}} engagement as a reference example of how a custom office/VC deployment was built. **Do NOT install this skill as part of a generic OpenClaw deployment.** It may contain engagement-specific defaults (emails, chat IDs, team names, VC-flavored workflows), Claude Opus routing (we default to Codex per `audit/_baseline.md` §BB in the parent repo), or stale "Blocked Commands" claims from 2026.2.26 that were resolved in 2026.4.15+. Use as inspiration when building custom-office skills, not as a default.

## Compatibility
- **OpenClaw Version**: 2026.3.22+
- **Status**: READY
- **Blocked Commands**: None
- **Notes**: This skill avoids all nonexistent OpenClaw CLI commands. Uses file-based config, Node.js scripts with raw API calls (Claude, Notion, Telegram), and launchd for any scheduling needs. Fully compatible with 2026.3.22 limitations.

## Purpose

Deploy a multi-person action item extraction, routing, and approval engine. When a meeting transcript arrives, the system uses Claude to extract structured action items, identifies the correct owner for each item, and routes approvals to the right person — not just {{EA_NAME}} for everything. Each action item goes to its owner for review, with {{AGENT_NAME}}'s Notion dashboard providing a Kanban-style view of all pending approvals across the team.

The system follows the Harness Pattern for ownership assignment: if {{AGENT_NAME}} isn't confident who should own an item, it does NOT guess. It flags it as "ownership unclear" on {{AGENT_NAME}}'s board and asks the people in the meeting for clarification.

**When to use:** When the client has meeting transcripts and needs action items extracted, routed to the correct owners, and tracked across a team — particularly for VC firms where meetings involve analysts, VPs, and executives who each have their own follow-ups.

**What this skill does:**
1. Configures action item extraction with multi-person routing rules
2. Installs `extract-actions.js` -- Claude-powered extraction with Jaro-Winkler fuzzy matching and ownership confidence scoring
3. Installs `route-actions.js` -- routes each item to the correct owner based on meeting context, role, and confidence
4. Installs `approval-flow.js` -- multi-channel approval (Telegram per person + {{AGENT_NAME}}'s Notion Kanban board)
5. Installs `process-meeting.js` -- orchestrator that chains extraction, routing, verification, approval, and Notion task creation
6. Creates {{AGENT_NAME}}'s Action Items Kanban board in Notion for team-wide visibility
7. Creates TOOLS.md entries so the agent can trigger the pipeline
8. Verifies end-to-end flow with multi-person sample transcript

## Multi-Person Routing System

### The Problem with "Everything Goes to {{EA_NAME}}"

{{EA_NAME}} manages {{PRINCIPAL_NAME}}'s calendar and personal operations, but she doesn't own analyst work. When {{TEAM_ANALYST}} and Gibson are on a call with {{PRINCIPAL_NAME}} and action items come out, each person has their own follow-ups:
- **{{PRINCIPAL_NAME}}**: "I'll send the metrics" → {{PRINCIPAL_NAME}}'s task
- **{{TEAM_ANALYST}}**: "I'll draft the board deck" → {{TEAM_ANALYST}}'s task
- **Gibson**: "I'll pull the comp data" → Gibson's task
- **{{EA_NAME}}**: "Schedule the follow-up call" → {{EA_NAME}}'s task (operational/calendar)

Routing everything through {{EA_NAME}} creates a bottleneck and means she's approving tasks she doesn't own or understand.

### Routing Rules

| Item Type | Who Owns It | Who Approves | How Detected |
|-----------|-------------|-------------|-------------|
| **{{PRINCIPAL_NAME}}'s personal follow-ups** | {{PRINCIPAL_NAME}} | {{EA_NAME}} reviews first (operational filter) | Owner = {{PRINCIPAL_NAME}} in transcript ("I'll...", "{{PRINCIPAL_NAME}} will...") |
| **Analyst work** (research, decks, data) | The analyst mentioned | The analyst themselves | Owner = {{TEAM_ANALYST}}/Gibson/etc. in transcript |
| **Senior leadership tasks** | {{TEAM_MEMBER_2}} / Joe | Themselves | Owner = {{TEAM_MEMBER_2}}/Joe in transcript |
| **{{EA_NAME}}'s operational tasks** | {{EA_NAME}} | {{EA_NAME}} (self-approve) | Scheduling, travel, communications tasks |
| **Investment-specific** | {{PRINCIPAL_NAME}} or {{TEAM_MEMBER_2}} | {{PRINCIPAL_NAME}} directly | Contains deal terms, portfolio, LP language |
| **Unclear ownership** | NOBODY — flagged on {{AGENT_NAME}}'s board | Meeting attendees asked for clarification | Ownership confidence < 0.7 (see below) |

### Ownership Confidence Scoring (Harness Pattern)

{{AGENT_NAME}} does NOT guess who owns a task. After Claude extracts an action item, an Ownership Verification Agent (Claude Sonnet) scores its confidence:

```
For each extracted action item:
  1. Who does the transcript say should do this?
     - Explicit: "{{TEAM_ANALYST}}, can you draft that?" → {{TEAM_ANALYST}}, confidence 0.95
     - Implicit: "We need someone to pull the data" → UNCLEAR, confidence 0.3
     - Ambiguous: "Let's get that done" → UNCLEAR, confidence 0.2

  2. Does the task type match the person's role?
     - Analyst doing research → HIGH confidence
     - Analyst making investment decisions → LOW confidence (probably {{PRINCIPAL_NAME}}'s)
     - {{EA_NAME}} doing scheduling → HIGH confidence
     - {{EA_NAME}} doing deal analysis → LOW confidence

  3. Confidence thresholds:
     ≥ 0.8: Route to identified owner for approval
     0.5-0.79: Route to identified owner BUT flag as "{{AGENT_NAME}} isn't 100% sure"
     < 0.5: DO NOT ASSIGN. Put on {{AGENT_NAME}}'s board as "Ownership unclear"
             and ask meeting attendees for clarification
```

### What Happens When {{AGENT_NAME}} Doesn't Know

When ownership confidence is below 0.5, the item goes to {{AGENT_NAME}}'s Notion "Needs Clarification" column:

```
{{AGENT_NAME}}'s Action Items Board → "Needs Clarification" column:

📋 "Pull comparative market data for Series B valuation"
   From: Series B Discussion (Mar 31, 9:00 AM)
   Attendees: {{PRINCIPAL_NAME}}, {{TEAM_ANALYST}}, Gibson

   {{AGENT_NAME}}'s note: "The transcript says 'we need to pull the data' but
   doesn't specify who. {{TEAM_ANALYST}} and Gibson are both analysts who could
   do this. I'm not confident enough to assign it."

   [Assign to {{TEAM_ANALYST}}] [Assign to Gibson] [Assign to {{PRINCIPAL_NAME}}] [Ask in meeting chat]
```

If nobody resolves it within 24 hours, {{AGENT_NAME}} sends a Telegram message to the meeting attendees:
```
From your Series B Discussion yesterday:
"Pull comparative market data for Series B valuation"

I wasn't sure who should own this. Can someone claim it?
[I'll do it — {{TEAM_ANALYST}}] [I'll do it — Gibson] [Assign to {{PRINCIPAL_NAME}}]
```

### {{AGENT_NAME}}'s Notion Action Items Dashboard (Kanban)

{{AGENT_NAME}} gets its own Notion database view — a Kanban board that shows all action items across the team:

```
┌─────────────────┬──────────────────┬──────────────────┬─────────────────┐
│ Needs            │ Pending          │ Approved /       │ Done            │
│ Clarification    │ Approval         │ In Progress      │                 │
├─────────────────┼──────────────────┼──────────────────┼─────────────────┤
│ 📋 Pull comp    │ 📋 Send metrics  │ 📋 Board deck   │ 📋 Schedule     │
│ data (WHO?)     │ → {{PRINCIPAL_NAME}} (waiting   │ → {{TEAM_ANALYST}}         │ follow-up call  │
│                 │   {{EA_NAME}} review)  │   (approved)     │ → {{EA_NAME}} ✓       │
│                 │                  │                  │                 │
│                 │ 📋 Draft email   │                  │                 │
│                 │ → Gibson         │                  │                 │
│                 │   (waiting)      │                  │                 │
└─────────────────┴──────────────────┴──────────────────┴─────────────────┘
```

This board is:
- **Created as a Notion database** with views (Kanban by status, filtered by owner, filtered by meeting)
- **Visible to the whole team** — {{TEAM_ANALYST}} can see his items, Gibson can see his, {{EA_NAME}} sees hers
- **Updated by {{AGENT_NAME}}** as approvals come in (via Notion oversight agent)
- **Phase-outable** — once the team trusts {{AGENT_NAME}}'s extractions, the "Pending Approval" step can be relaxed to auto-approve high-confidence items, with the board serving as a review-if-needed view instead of a gate

### Per-Person Approval Channels

Each CVT team member gets their action items in their own channel:

| Person | Approval Channel | What They See |
|--------|-----------------|---------------|
| **{{PRINCIPAL_NAME}}** | Telegram DM (investment items) or via {{EA_NAME}} (operational) | Only items he owns or needs to decide on |
| **{{EA_NAME}}** | Telegram DM | {{PRINCIPAL_NAME}}'s operational items (she reviews first) + her own items |
| **{{TEAM_ANALYST}}** | Telegram DM (once set up) or {{AGENT_NAME}}'s Notion board | Analyst tasks he owns |
| **Gibson** | Telegram DM (once set up) or {{AGENT_NAME}}'s Notion board | Analyst tasks he owns |
| **{{TEAM_MEMBER_2}}** | Telegram DM (once set up) or {{AGENT_NAME}}'s Notion board | VP tasks he owns |
| **Joe** | Telegram DM (once set up) or {{AGENT_NAME}}'s Notion board | VP tasks he owns |

**If a team member isn't on Telegram**, their items go to {{AGENT_NAME}}'s Notion board where they can approve by moving the card from "Pending Approval" to "Approved."

### Phased Trust Model

This approval system is designed to be **phased out** as {{AGENT_NAME}} earns trust:

| Phase | When | What Changes |
|-------|------|-------------|
| **Phase 1: Full approval** | Day 1 | Every item requires human approval. Nothing auto-created. |
| **Phase 2: High-confidence auto-approve** | After ~2 weeks of accurate extractions | Items with ownership confidence ≥ 0.9 AND task type matches role → auto-created in Notion. Still visible on board. |
| **Phase 3: Review-if-needed** | After ~1 month | All items auto-created. Board serves as review view. Humans fix wrong items rather than approving every correct one. |
| **Phase 4: Exception-only** | When trust is established | Only "unclear ownership" items need human input. Everything else flows automatically. |

{{EA_NAME}} and {{PRINCIPAL_NAME}} decide when to advance phases. {{AGENT_NAME}} tracks its accuracy over time (% of items approved without modification) to support the decision.

## How Commands Are Sent

This skill is executed by an operator who has an active remote session.
See `_deploy-common.md` -> "Command Delivery" for the full protocol.

- **Remote:** commands sent via `POST /api/devices/${DEVICE_ID}/exec` (preferred for enrolled devices) or `POST /api/sessions/${SESSION_ID}/commands`
- **Operator:** commands run on the operator's local terminal

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${DEVICE_ID}` | Device enrollment or session context | `dev_abc123` |
| `${SESSION_ID}` | Session polling or creation (fallback if not enrolled) | `sess_abc123` |
| `${BEARER}` | `POST /api/operator/login` response | `eyJhbG...` |
| `${NOTION_API_KEY}` | Client: Notion Settings > Integrations > Internal integration token | `ntn_abc123def456...` |
| `${NOTION_TASKS_DB_ID}` | Notion config or client provides: Tasks/Action Items database | `c3d4e5f6a1b2...` |
| `${NOTION_PEOPLE_DB_ID}` | Notion config or client provides: People database | `a1b2c3d4e5f6...` |
| `${NOTION_COMPANIES_DB_ID}` | Notion config or client provides: Companies database | `b2c3d4e5f6a1...` |
| `${ANTHROPIC_API_KEY}` | Client profile or `.env`: Claude API key | `sk-ant-api03-...` |
| `${TELEGRAM_BOT_TOKEN}` | Client profile: agent's Telegram bot token | `7123456789:AAF...` |
| `${TELEGRAM_CHAT_ID}` | Client profile: client's Telegram user ID | `{{PRINCIPAL_CHAT_ID}}` |
| `${TIMEZONE}` | Client profile: timezone | `America/Denver` |
| `${AGENT_USER}` | Pre-flight check: `whoami` on client machine | `edge` |
| `${AGENT_USER_UID}` | Pre-flight check: `id -u` on client machine | `501` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if Node.js is available (required for all scripts):

**Remote:**
```
node --version 2>/dev/null || echo 'NO_NODE'
```

Expected: Version string (v18+ required for native fetch). If `NO_NODE`, install Node.js first.

Check if Notion workspace integration is already deployed:

**Remote:**
```
test -f ${WORKSPACE}/config/notion.json && echo 'NOTION_CONFIGURED' || echo 'NO_NOTION'
```

Expected: `NOTION_CONFIGURED`. If `NO_NOTION`, deploy `deploy-notion-workspace.md` first -- this skill depends on the Notion config for database access.

Check if action items config already exists:

**Remote:**
```
test -f ${WORKSPACE}/config/action-items.json && cat ${WORKSPACE}/config/action-items.json | head -20 || echo 'NOT_CONFIGURED'
```

Expected: Either existing config (review for correctness) or `NOT_CONFIGURED`.

Check if action item scripts already exist:

**Remote:**
```
ls ${WORKSPACE}/tools/extract-actions.js ${WORKSPACE}/tools/approval-flow.js ${WORKSPACE}/tools/process-meeting.js 2>/dev/null || echo 'NO_SCRIPTS'
```

Expected: Either file paths listed (review for correctness) or `NO_SCRIPTS`.

Resolve the macOS username and UID (needed for absolute paths in scripts and any launchd plists):

**Remote:**
```
whoami && id -u
```

Expected: Username and numeric UID. Record both.

Verify the Claude API key is available and valid:

**Remote:**
```
grep ANTHROPIC_API_KEY ${WORKSPACE}/.env 2>/dev/null | head -c 30 || echo 'NO_ANTHROPIC_KEY'
```

Expected: Shows `ANTHROPIC_API_KEY=sk-ant-api03-` prefix. If missing, the client must provide a Claude API key.

Verify the Telegram bot token is available:

**Remote:**
```
grep TELEGRAM_BOT_TOKEN ${WORKSPACE}/.env 2>/dev/null | head -c 30 || echo 'NO_BOT_TOKEN'
```

Expected: Shows `TELEGRAM_BOT_TOKEN=` with a value. If missing, set up messaging first via `deploy-messaging-setup.md`.

**Decision points from pre-flight:**
- Is `deploy-notion-workspace.md` completed? The action item engine writes to the Tasks database and reads from People/Companies for fuzzy matching. This is a hard dependency.
- Does the client have a Claude API key? The extraction step requires the Anthropic API.
- Is Telegram messaging set up? The approval flow sends inline keyboard messages to the client.
- What are the valid status values in the Tasks database? The approval flow sets status to "Pending Approval" and "To Do" -- confirm these exist as options.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Extraction model | `claude-sonnet-4-20250514` | Client wants higher quality (Opus) or lower cost (Haiku) |
| Fuzzy match threshold | 0.85 Jaro-Winkler score | Client has many similar names (lower to 0.80) or wants stricter matching (raise to 0.90) |
| Approval timeout phase 1 | 15 minutes | Client responds slowly (increase) or wants faster escalation (decrease) |
| Approval timeout phase 2 | 15 minutes (30 min total) | Adjust total wait before escalation |
| Pending status value | `Pending Approval` | Client's Tasks DB uses different status options (e.g., "Draft", "Needs Review") |
| Approved status value | `To Do` | Client uses "Backlog", "Ready", "Open", etc. |
| Rejected status behavior | Archive (soft delete) in Notion | Client wants rejected items kept visible with a "Rejected" status instead |
| Transcript input path | `${WORKSPACE}/data/transcripts/` | Client stores transcripts elsewhere |
| Action item JSON schema | Default 7 fields (description, owner, due_date, priority, related_company, related_people, context) | Client needs additional fields (e.g., project, tags, effort estimate) |
| Audit log location | `${WORKSPACE}/logs/action-items-audit.log` | Client wants audit logs in a different path or format |
| People DB page limit for matching | 500 cached records | Adjust for very large (10,000+) or small (<100) People databases |
| Config location | `${WORKSPACE}/config/action-items.json` | Client prefers config in a different path |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- Node.js v18+ available on the client machine
- `deploy-identity.md` completed (agent identity configured)
- `deploy-messaging-setup.md` completed (Telegram bot configured and working)
- `deploy-notion-workspace.md` completed (Notion config, People/Companies/Tasks databases mapped)
- Claude API key available (`sk-ant-api03-` prefix validated)
- Telegram bot token and client chat ID available

## What Gets Installed

### Action Items Config (`config/action-items.json`)

| Field | Description |
|-------|-------------|
| `anthropic.apiKey` | Reference to env var for Claude API key |
| `anthropic.model` | Model for extraction (default: `claude-sonnet-4-20250514`) |
| `anthropic.maxTokens` | Max output tokens for extraction (default: 4096) |
| `notion.configPath` | Path to `notion.json` for database access |
| `notion.tasksDbId` | Tasks/Action Items database ID |
| `notion.peopleDbId` | People database ID |
| `notion.companiesDbId` | Companies database ID |
| `notion.statusPending` | Status value for unapproved items (default: "Pending Approval") |
| `notion.statusApproved` | Status value for approved items (default: "To Do") |
| `telegram.botToken` | Reference to env var for bot token |
| `telegram.chatId` | Client's Telegram user ID |
| `matching.algorithm` | Fuzzy match algorithm (default: "jaro-winkler") |
| `matching.threshold` | Minimum score for a match (default: 0.85) |
| `matching.cacheMaxAge` | Max age of People/Companies cache for matching (default: 3600000 ms) |
| `approval.timeoutPhase1` | First timeout in ms (default: 900000 = 15 min) |
| `approval.timeoutPhase2` | Second timeout in ms (default: 900000 = 15 min) |
| `paths.transcripts` | Input directory for transcripts |
| `paths.auditLog` | Audit log file path |

### Scripts

| Script | Location | Purpose |
|--------|----------|---------|
| `extract-actions.js` | `${WORKSPACE}/tools/extract-actions.js` | Claude-powered action item extraction with fuzzy name matching |
| `approval-flow.js` | `${WORKSPACE}/tools/approval-flow.js` | Telegram inline keyboard approval with timeout and escalation |
| `process-meeting.js` | `${WORKSPACE}/tools/process-meeting.js` | Orchestrator: transcript in -> approved tasks out |

### Audit Log

| Detail | Value |
|--------|-------|
| Location | `${WORKSPACE}/logs/action-items-audit.log` |
| Format | JSONL (one JSON object per line) |
| Fields per entry | `timestamp`, `action`, `item_description`, `decision`, `decided_by`, `notion_page_id`, `reason` |

### TOOLS.md Entries

Adds entries to `${WORKSPACE}/TOOLS.md` so the agent knows:
- How to process a meeting transcript for action items
- How to check pending approval status
- How to manually trigger extraction on a text file

## Steps

### Phase 1: Configuration

#### 1.1 Create Directory Structure `[AUTO]`

Create all directories needed for the action item pipeline.

**Remote:**
```
mkdir -p ${WORKSPACE}/config ${WORKSPACE}/tools ${WORKSPACE}/logs ${WORKSPACE}/data/transcripts ${WORKSPACE}/data/action-items
```

Expected: All directories exist. `mkdir -p` is idempotent.

#### 1.2 Store API Keys `[HUMAN_INPUT]`

Ensure the Claude API key and Telegram bot token are present in the workspace `.env` file. These may already exist from prior skill deployments.

Check and add the Claude API key if missing:

**Remote:**
```
grep -q ANTHROPIC_API_KEY ${WORKSPACE}/.env 2>/dev/null && echo 'ALREADY_SET' || echo 'ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}' >> ${WORKSPACE}/.env
```

Expected: Line present in `.env`. Verify:

**Remote:**
```
grep ANTHROPIC_API_KEY ${WORKSPACE}/.env | head -c 30
```

Expected: Shows `ANTHROPIC_API_KEY=sk-ant-api03-` prefix.

Check and add the Telegram bot token if missing:

**Remote:**
```
grep -q TELEGRAM_BOT_TOKEN ${WORKSPACE}/.env 2>/dev/null && echo 'ALREADY_SET' || echo 'TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}' >> ${WORKSPACE}/.env
```

Check and add the Telegram chat ID if missing:

**Remote:**
```
grep -q TELEGRAM_CHAT_ID ${WORKSPACE}/.env 2>/dev/null && echo 'ALREADY_SET' || echo 'TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}' >> ${WORKSPACE}/.env
```

If any key is missing and not provided by prior skills, pause and collect from the client. The Claude API key **must** start with `sk-ant-api03-`. If it does not, STOP and ask for the correct key.

If already exists: Verify values are still valid by testing in Phase 4.

#### 1.3 Read Notion Database IDs from Existing Config `[AUTO]`

Pull the database IDs from the existing Notion config rather than asking the client again.

**Remote:**
```
python3 -c "
import json
c = json.load(open('${WORKSPACE}/config/notion.json'))
dbs = c.get('databases', {})
print('TASKS_DB:', dbs.get('tasks', {}).get('id', 'MISSING'))
print('PEOPLE_DB:', dbs.get('people', {}).get('id', 'MISSING'))
print('COMPANIES_DB:', dbs.get('companies', {}).get('id', 'MISSING'))
"
```

Expected: Three database IDs printed. If any says `MISSING`, the Notion workspace skill was not fully deployed -- deploy it first.

Record these as `${NOTION_TASKS_DB_ID}`, `${NOTION_PEOPLE_DB_ID}`, and `${NOTION_COMPANIES_DB_ID}` for use in the config.

#### 1.4 Discover Valid Task Status Options `[GUIDED]`

Query the Tasks database schema to find the valid status options. The approval flow needs to set "Pending Approval" and "To Do" -- these must exist as valid status values.

**Remote:**
```
curl -s "https://api.notion.com/v1/databases/${NOTION_TASKS_DB_ID}" \
  -H "Authorization: Bearer ${NOTION_API_KEY}" \
  -H "Notion-Version: 2022-06-28" | python3 -c "
import json, sys
db = json.load(sys.stdin)
props = db.get('properties', {})
for name, prop in props.items():
    if prop['type'] in ('status', 'select'):
        print(f'Property: {name} (type: {prop[\"type\"]})')
        if prop['type'] == 'status':
            for group in prop.get('status', {}).get('groups', []):
                for opt in group.get('options', []):
                    print(f'  - {opt[\"name\"]} ({group[\"name\"]})')
        elif prop['type'] == 'select':
            for opt in prop.get('select', {}).get('options', []):
                print(f'  - {opt[\"name\"]}')
"
```

Expected: List of valid status/select values for the Tasks database. Record which values to use for "pending" and "approved" states.

If "Pending Approval" does not exist as a status option:
- If the status property is a `status` type: Notion auto-creates new status values on first use. Proceed -- the approval flow will create it.
- If the status property is a `select` type: New values can be created on first use. Proceed.
- Present the available options to the operator and confirm the mapping before proceeding.

#### 1.5 Write Action Items Config `[GUIDED]`

Write the config file that the extraction and approval scripts will read.

**Remote (use base64 encoding per `_deploy-common.md` File Transfer Standard):**

Encode the JSON config content as base64 on the operator side, then send:

```
echo '<base64-encoded-action-items-config-json>' | base64 -d > ${WORKSPACE}/config/action-items.json
```

The JSON structure must follow this template:

```json
{
  "anthropic": {
    "apiKeyEnv": "ANTHROPIC_API_KEY",
    "model": "claude-sonnet-4-20250514",
    "maxTokens": 4096
  },
  "notion": {
    "configPath": "./config/notion.json",
    "tasksDbId": "${NOTION_TASKS_DB_ID}",
    "peopleDbId": "${NOTION_PEOPLE_DB_ID}",
    "companiesDbId": "${NOTION_COMPANIES_DB_ID}",
    "statusPending": "Pending Approval",
    "statusApproved": "To Do"
  },
  "telegram": {
    "botTokenEnv": "TELEGRAM_BOT_TOKEN",
    "chatId": "${TELEGRAM_CHAT_ID}"
  },
  "matching": {
    "algorithm": "jaro-winkler",
    "threshold": 0.85,
    "cacheMaxAgeMs": 3600000
  },
  "approval": {
    "timeoutPhase1Ms": 900000,
    "timeoutPhase2Ms": 900000
  },
  "extraction": {
    "schema": {
      "description": "string - what needs to be done",
      "owner": "string - person responsible (name as mentioned in transcript)",
      "due_date": "string|null - YYYY-MM-DD or null if not mentioned",
      "priority": "string - high|medium|low (inferred from urgency cues)",
      "related_company": "string|null - company name if mentioned",
      "related_people": "array - other people involved or mentioned",
      "context": "string - surrounding context from the transcript"
    }
  },
  "paths": {
    "transcripts": "./data/transcripts",
    "actionItems": "./data/action-items",
    "auditLog": "./logs/action-items-audit.log"
  }
}
```

**Critical:** The `statusPending` and `statusApproved` values must match valid status options discovered in step 1.4. If the client's Tasks database uses different terminology, adapt these values.

Expected: File exists at `${WORKSPACE}/config/action-items.json` with valid JSON.

If this fails: Check that `${WORKSPACE}/config/` exists. Verify base64 encoding produced valid JSON with `python3 -c "import json; json.load(open('${WORKSPACE}/config/action-items.json'))"`.

If already exists: Compare content. If unchanged, skip. If different, back up as `action-items.json.bak` and write new version.

### Phase 2: Install Extraction Script

#### 2.1 Install extract-actions.js `[GUIDED]`

Write the extraction script that uses Claude to parse meeting transcripts into structured action items, then fuzzy-matches mentioned people and companies against Notion databases.

The script must:
- Read config from `${WORKSPACE}/config/action-items.json`
- Load the Notion config from the path specified in `notion.configPath`
- Accept input as a file path argument or stdin
- Read the `.env` file from the workspace root to resolve API keys referenced by env var name
- Call the Claude API (`POST https://api.anthropic.com/v1/messages`) with a structured extraction prompt
- Use tool_use/function calling to get structured JSON output matching the extraction schema
- After extraction, load People and Companies from Notion (cached for `cacheMaxAgeMs`)
- For each `owner` and `related_people` name, run Jaro-Winkler fuzzy matching against cached People names
- For each `related_company`, run Jaro-Winkler fuzzy matching against cached Companies names
- Attach matched Notion page IDs to each action item for downstream linking
- If a name scores below the threshold, mark it as `unmatched` with the best candidate and score
- Output structured JSON array to stdout
- Support `--verbose` flag for detailed matching output
- Support `--dry-run` flag to show the Claude prompt without making the API call
- Exit with code 0 on success, non-zero on error with descriptive message to stderr

**Jaro-Winkler implementation requirements:**
- Implement Jaro-Winkler directly in the script (pure JavaScript, no external dependency)
- Normalize names before comparison: lowercase, trim whitespace, remove common prefixes ("dr.", "mr.", "mrs.")
- Support partial matching: "{{PRINCIPAL_NAME}}" should match "{{PRINCIPAL_NAME}}" (compare against first name, last name, and full name)
- Return the best match above threshold, or null if no match meets the threshold

**Claude prompt design requirements:**
- System prompt must explain the context: extracting action items from a meeting transcript for a VC executive
- Include the JSON schema in the prompt so Claude returns structured data
- Instruct Claude to infer priority from urgency language ("ASAP", "by end of week", "when you get a chance")
- Instruct Claude to infer due dates from relative time references ("next Tuesday", "by Friday") resolved against today's date in `${TIMEZONE}`
- Instruct Claude to distinguish between action items (things someone committed to do) and discussion points (things that were talked about but no action assigned)
- Use `temperature: 0` for deterministic extraction

**Example input/output:**

Input transcript:
```
{{PRINCIPAL_NAME}}: Let's get a term sheet to Acme Corp by Friday. Sarah, can you handle the due diligence?
Sarah: Sure, I'll start the background check tomorrow.
{{PRINCIPAL_NAME}}: Great. Also, someone should follow up with John Park about the Series B timeline.
```

Expected output:
```json
[
  {
    "description": "Send term sheet to Acme Corp",
    "owner": "{{PRINCIPAL_NAME}}",
    "due_date": "2026-03-31",
    "priority": "high",
    "related_company": "Acme Corp",
    "related_people": ["Sarah"],
    "context": "{{PRINCIPAL_NAME}} committed to getting a term sheet to Acme Corp by Friday",
    "matched_owner": {"name": "{{PRINCIPAL_NAME}}", "notion_id": "abc123", "score": 0.92},
    "matched_company": {"name": "Acme Corporation", "notion_id": "def456", "score": 0.88},
    "matched_people": [{"name": "{{EXTERNAL_CONTACT_1}}", "notion_id": "ghi789", "score": 0.95}]
  },
  {
    "description": "Conduct due diligence / background check on Acme Corp",
    "owner": "Sarah",
    "due_date": "2026-03-28",
    "priority": "high",
    "related_company": "Acme Corp",
    "related_people": ["{{PRINCIPAL_NAME}}"],
    "context": "Sarah volunteered to start the background check tomorrow",
    "matched_owner": {"name": "{{EXTERNAL_CONTACT_1}}", "notion_id": "ghi789", "score": 0.95},
    "matched_company": {"name": "Acme Corporation", "notion_id": "def456", "score": 0.88},
    "matched_people": [{"name": "{{PRINCIPAL_NAME}}", "notion_id": "abc123", "score": 0.92}]
  },
  {
    "description": "Follow up with John Park about Series B timeline",
    "owner": null,
    "due_date": null,
    "priority": "medium",
    "related_company": null,
    "related_people": ["John Park"],
    "context": "{{PRINCIPAL_NAME}} mentioned someone should follow up, but no specific owner was assigned",
    "matched_owner": null,
    "matched_people": [{"name": "John Park", "notion_id": "jkl012", "score": 1.0}]
  }
]
```

Write the script to `${WORKSPACE}/tools/extract-actions.js` using base64 encoding.

**Remote:**
```
echo '<base64-encoded-extract-actions-js>' | base64 -d > ${WORKSPACE}/tools/extract-actions.js
```

Expected: File exists at `${WORKSPACE}/tools/extract-actions.js`. Test with `node ${WORKSPACE}/tools/extract-actions.js --dry-run < /dev/null` -- should show the system prompt without making an API call.

If this fails: Check Node.js version (v18+ for native fetch). If older, the script must use the `https` module instead of `fetch`.

If already exists: Compare content. If unchanged, skip. If different, back up as `extract-actions.js.bak` and write new version.

### Phase 3: Install Approval Flow Script

#### 3.1 Install approval-flow.js `[GUIDED]`

Write the approval flow script that sends extracted action items to the client via Telegram for individual approve/reject decisions using inline keyboard buttons.

The script must:
- Read config from `${WORKSPACE}/config/action-items.json`
- Read the `.env` file from the workspace root to resolve bot token
- Accept extracted action items JSON as input (file path argument or stdin)
- For each action item:
  1. Create a draft task in Notion Tasks database with status set to `statusPending`
  2. Format a Telegram message with: description, owner, due date, priority, company, and context
  3. Send the message with inline keyboard buttons: "Approve", "Reject", "Edit"
- Handle Telegram webhook/polling conflict per `_deploy-common.md` -> "Telegram Client Interaction":
  1. Check if a webhook is set with `getWebhookInfo`
  2. If set, save and temporarily remove it
  3. Set EXIT trap to restore webhook on any exit
  4. Use `getUpdates` with long polling for callback queries
- Poll for callback query responses (inline button presses) from the specific `${TELEGRAM_CHAT_ID}`:
  - **Phase 1:** Poll for `timeoutPhase1Ms` (default 15 minutes)
  - **On phase 1 timeout:** Send a gentle reminder and continue polling
  - **Phase 2:** Poll for `timeoutPhase2Ms` (default 15 more minutes)
  - **On phase 2 timeout:** Log escalation, save pending state for later resume
- Process each callback:
  - **Approve:** Update Notion task status from `statusPending` to `statusApproved`. Answer callback with confirmation.
  - **Reject:** Send follow-up message asking for rejection reason (optional). Archive the Notion task. Answer callback with confirmation.
  - **Edit:** Send follow-up message asking what to change. Apply edits to the Notion task. Re-send for approval.
- After all items are decided (or timeout):
  - Restore Telegram webhook if one was saved
  - Write audit log entries (JSONL format) for every decision
  - Output summary JSON to stdout: total items, approved count, rejected count, timed out count
- Support `--resume` flag to pick up undecided items from a previous interrupted run
- Exit with code 0 on success, non-zero on error

**Telegram inline keyboard format:**

Each action item message should look like:

```
New Action Item (1/3)

Task: Send term sheet to Acme Corp
Owner: {{PRINCIPAL_NAME}}
Due: Mar 31, 2026
Priority: HIGH
Company: Acme Corporation
Context: {{PRINCIPAL_NAME}} committed to getting a term sheet to Acme Corp by Friday

[Approve] [Reject] [Edit]
```

**Telegram inline keyboard API usage:**

```json
{
  "chat_id": "${TELEGRAM_CHAT_ID}",
  "text": "...",
  "parse_mode": "Markdown",
  "reply_markup": {
    "inline_keyboard": [[
      {"text": "Approve", "callback_data": "approve_ITEM_ID"},
      {"text": "Reject", "callback_data": "reject_ITEM_ID"},
      {"text": "Edit", "callback_data": "edit_ITEM_ID"}
    ]]
  }
}
```

**Callback query handling:**

Poll for callback queries using `getUpdates` with `allowed_updates: ["callback_query"]`. When a callback is received:
- Parse `callback_data` to get the action and item ID
- Call `answerCallbackQuery` to dismiss the loading indicator
- Execute the corresponding action (approve/reject/edit)
- Update the original message to show the decision (e.g., strike through rejected items, add checkmark for approved)

**Audit log format (JSONL):**

Each line is a JSON object:
```json
{"timestamp":"2026-03-27T14:30:00-06:00","action":"approve","item_description":"Send term sheet to Acme Corp","decision":"approved","decided_by":"principal","notion_page_id":"page_abc123","reason":null}
```

Write the script to `${WORKSPACE}/tools/approval-flow.js` using base64 encoding.

**Remote:**
```
echo '<base64-encoded-approval-flow-js>' | base64 -d > ${WORKSPACE}/tools/approval-flow.js
```

Expected: File exists at `${WORKSPACE}/tools/approval-flow.js`.

If this fails: Check Node.js version and config file. Verify Telegram bot token is valid with a `getMe` call.

If already exists: Compare content. If unchanged, skip. If different, back up as `approval-flow.js.bak` and write new version.

### Phase 4: Install Orchestrator Script

#### 4.1 Install process-meeting.js `[GUIDED]`

Write the orchestrator script that chains the full pipeline: read transcript, extract action items, run approval flow, generate summary.

The script must:
- Read config from `${WORKSPACE}/config/action-items.json`
- Accept a transcript file path as argument, or read from stdin
- Support `--transcript PATH` flag for explicit file path
- Support `--text "INLINE_TEXT"` flag for short inline transcripts
- Support `--date YYYY-MM-DD` flag to set the reference date for relative date resolution (defaults to today in `${TIMEZONE}`)
- Execute the pipeline:
  1. Read and validate the transcript (must be non-empty, minimum 50 characters)
  2. Call `extract-actions.js` as a child process, pipe transcript as stdin
  3. Parse the extraction output JSON
  4. If zero action items extracted, send a Telegram message: "No action items found in this transcript" and exit
  5. Call `approval-flow.js` as a child process, pipe extracted items as stdin
  6. Parse the approval output JSON (summary of decisions)
  7. Generate a summary report
- Send a final summary to Telegram:
  ```
  Meeting Processing Complete

  Transcript: [filename or "inline"]
  Action items found: 5
  Approved: 3
  Rejected: 1
  Timed out: 1

  Approved items have been added to your Tasks.
  ```
- Save the full pipeline output (extraction + decisions + summary) to `${WORKSPACE}/data/action-items/YYYY-MM-DD_HHmmss.json`
- Log the run to the audit log
- Exit with code 0 on success, non-zero on error

Write the script to `${WORKSPACE}/tools/process-meeting.js` using base64 encoding.

**Remote:**
```
echo '<base64-encoded-process-meeting-js>' | base64 -d > ${WORKSPACE}/tools/process-meeting.js
```

Expected: File exists at `${WORKSPACE}/tools/process-meeting.js`.

If this fails: Check that `extract-actions.js` and `approval-flow.js` exist at the expected paths.

If already exists: Compare content. If unchanged, skip. If different, back up and write new version.

### Phase 5: TOOLS.md Integration

#### 5.1 Write TOOLS.md Entries `[GUIDED]`

Add entries to `${WORKSPACE}/TOOLS.md` so the agent knows how to trigger the action item pipeline. These entries define what the agent can invoke during conversations.

Check if TOOLS.md exists:

**Remote:**
```
test -f ${WORKSPACE}/TOOLS.md && echo 'EXISTS' || echo 'NOT_FOUND'
```

If it exists, append the Action Items section. If not, create it with the Action Items section.

Check if an Action Items section already exists:

**Remote:**
```
grep -q "## Action Item Extraction" ${WORKSPACE}/TOOLS.md 2>/dev/null && echo 'SECTION_EXISTS' || echo 'NO_SECTION'
```

The Action Items TOOLS.md content to add:

```markdown
## Action Item Extraction

### Process Meeting Transcript
Extract action items from a meeting transcript, get {{PRINCIPAL_NAME}}'s approval via Telegram, and create approved tasks in Notion.
```
node ~/clawd/tools/process-meeting.js --transcript PATH_TO_TRANSCRIPT
```
Use for: after any meeting, when {{PRINCIPAL_NAME}} says "process the meeting notes", when a new transcript arrives.
Triggers approval flow -- {{PRINCIPAL_NAME}} will receive Telegram messages with Approve/Reject buttons for each item.

### Extract Action Items Only (No Approval)
Extract action items from text without triggering the approval flow. Useful for previewing what would be extracted.
```
node ~/clawd/tools/extract-actions.js PATH_TO_TRANSCRIPT
```
Use for: previewing extraction before sending for approval, debugging extraction quality.

### Process Inline Text
Extract action items from text provided directly (for short notes or verbal instructions).
```
node ~/clawd/tools/process-meeting.js --text "{{PRINCIPAL_NAME}} said to follow up with Acme Corp by Friday about the term sheet."
```
Use for: when {{PRINCIPAL_NAME}} dictates action items verbally or sends short notes.

### Check Pending Approvals
Resume approval flow for any items that timed out in a previous run.
```
node ~/clawd/tools/approval-flow.js --resume
```
Use for: when {{PRINCIPAL_NAME}} didn't respond to approval requests in time and wants to review them later.
```

Write the TOOLS.md content using base64 encoding.

If TOOLS.md already exists with no Action Items section, **append** the section:

**Remote:**
```
echo '<base64-encoded-action-items-tools-section>' | base64 -d >> ${WORKSPACE}/TOOLS.md
```

If `SECTION_EXISTS`, compare and update if different. If no TOOLS.md at all, create it:

**Remote:**
```
echo '<base64-encoded-action-items-tools-section>' | base64 -d > ${WORKSPACE}/TOOLS.md
```

Expected: TOOLS.md contains the "Action Item Extraction" section with all tool entries.

If already exists with correct content: Skip.

### Phase 6: Verification

#### 6.1 Validate Config Files `[AUTO]`

Verify all config files are valid and contain expected values.

**Remote:**
```
cd ${WORKSPACE} && python3 -c "
import json
# Action items config
c = json.load(open('config/action-items.json'))
print('Model:', c['anthropic']['model'])
print('Tasks DB:', c['notion']['tasksDbId'][:8] + '...')
print('People DB:', c['notion']['peopleDbId'][:8] + '...')
print('Companies DB:', c['notion']['companiesDbId'][:8] + '...')
print('Chat ID:', c['telegram']['chatId'])
print('Match threshold:', c['matching']['threshold'])
print('Config OK')
"
```

Expected: Shows model, truncated database IDs, chat ID, threshold, and "Config OK".

#### 6.2 Validate Claude API Connection `[AUTO]`

Test that the Claude API key works with a minimal request.

**Remote:**
```
cd ${WORKSPACE} && node -e "
const fs = require('fs');
const dotenv = {};
fs.readFileSync('.env', 'utf8').split('\n').forEach(line => {
  const [k, ...v] = line.split('=');
  if (k && v.length) dotenv[k.trim()] = v.join('=').trim();
});
fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'x-api-key': dotenv.ANTHROPIC_API_KEY,
    'anthropic-version': '2023-06-01',
    'content-type': 'application/json'
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 50,
    messages: [{role: 'user', content: 'Say OK'}]
  })
}).then(r => {
  console.log('Status:', r.status);
  if (r.status === 200) console.log('Claude API OK');
  else r.text().then(t => console.log('Error:', t));
}).catch(e => console.log('Network error:', e.message));
"
```

Expected: `Status: 200` and `Claude API OK`.

If this fails:
- **401:** API key is invalid. Check the key prefix (`sk-ant-api03-`). Ask the client for a valid key.
- **403:** Key is valid but may be rate-limited or restricted.
- **Network error:** Check internet connectivity on the client machine.

#### 6.3 Validate Telegram Bot `[AUTO]`

Test that the Telegram bot token is valid and can reach the client.

**Remote:**
```
cd ${WORKSPACE} && node -e "
const fs = require('fs');
const dotenv = {};
fs.readFileSync('.env', 'utf8').split('\n').forEach(line => {
  const [k, ...v] = line.split('=');
  if (k && v.length) dotenv[k.trim()] = v.join('=').trim();
});
fetch('https://api.telegram.org/bot' + dotenv.TELEGRAM_BOT_TOKEN + '/getMe')
  .then(r => r.json())
  .then(d => {
    if (d.ok) console.log('Bot OK:', d.result.username);
    else console.log('Bot error:', d.description);
  }).catch(e => console.log('Network error:', e.message));
"
```

Expected: `Bot OK: <bot_username>`.

If this fails:
- **401 Unauthorized:** Bot token is invalid. Check the token format and value.
- **Network error:** Check internet connectivity.

#### 6.4 Test Extraction with Sample Transcript `[GUIDED]`

Create a sample transcript and run extraction to verify the full extraction pipeline works.

**Remote:**
```
cat > ${WORKSPACE}/data/transcripts/test-transcript.txt << 'SAMPLE_EOF'
Meeting: Weekly Portfolio Review
Date: March 27, 2026
Participants: {{PRINCIPAL_NAME}}, {{EXTERNAL_CONTACT_1}}, {{EXTERNAL_CONTACT_2}}

{{PRINCIPAL_NAME}}: Good morning everyone. First item -- we need to close the Meridian deal. Sarah, can you send the updated term sheet by end of day tomorrow?

Sarah: Absolutely, I'll have it ready. Should I cc legal?

{{PRINCIPAL_NAME}}: Yes, loop in Martinez from Wilson & Hart. Also, Mike, can you pull the latest financials for Apex Dynamics? I want to review them before our board meeting next Tuesday.

Mike: On it. I'll have the analysis ready by Monday.

{{PRINCIPAL_NAME}}: Perfect. One more thing -- someone needs to schedule a follow-up call with the Pinnacle team. They've been waiting on our feedback for a week now. Let's make that happen by Friday.

Sarah: I can handle that. I'll reach out to their CEO directly.

{{PRINCIPAL_NAME}}: Great. That's it for today. Thanks team.
SAMPLE_EOF
```

Expected: File created at `${WORKSPACE}/data/transcripts/test-transcript.txt`.

Now run extraction (without approval flow):

**Remote:**
```
cd ${WORKSPACE} && node tools/extract-actions.js data/transcripts/test-transcript.txt
```

Expected: JSON array of 4-5 action items extracted:
1. Sarah: Send updated term sheet for Meridian deal (due: tomorrow)
2. Sarah: CC / loop in Martinez from Wilson & Hart on the term sheet
3. Mike: Pull latest financials for Apex Dynamics (due: Monday)
4. Sarah: Schedule follow-up call with Pinnacle team (due: Friday)

Each item should include `matched_owner`, `matched_company`, and `matched_people` fields showing fuzzy match results against Notion databases. Unmatched names will show as `null` with best candidate info.

If extraction fails:
- **Claude API error:** Check API key validity (step 6.2).
- **Empty results:** Review the system prompt in the script. Ensure it instructs Claude to extract action items specifically.
- **Malformed JSON:** Check that the script properly parses Claude's response. The extraction should use tool_use or a JSON-instructed response format.
- **No fuzzy matches:** This is OK for a test if the sample names don't exist in the Notion People/Companies databases. The matching is working correctly if it reports `unmatched` with scores.

#### 6.5 Test Full Pipeline with Approval `[GUIDED]`

Run the complete pipeline including Telegram approval. **This will send real messages to the client's Telegram.** Confirm with the operator before proceeding.

> **CONFIRM:** This will send Telegram messages with inline keyboard buttons to {{PRINCIPAL_NAME}} (chat ID ${TELEGRAM_CHAT_ID}). The operator should alert {{PRINCIPAL_NAME}} that a test approval flow is about to arrive. Proceed?

**Remote:**
```
cd ${WORKSPACE} && node tools/process-meeting.js --transcript data/transcripts/test-transcript.txt
```

Expected behavior:
1. Extraction runs and finds action items
2. Draft tasks created in Notion Tasks database with "Pending Approval" status
3. Telegram messages sent to {{PRINCIPAL_NAME}} with inline keyboard buttons (Approve/Reject/Edit)
4. Script waits for {{PRINCIPAL_NAME}}'s responses (up to 30 minutes with reminder at 15 min)
5. On {{PRINCIPAL_NAME}}'s approval: Notion task status updated to "To Do"
6. On {{PRINCIPAL_NAME}}'s rejection: Notion task archived
7. Summary message sent to {{PRINCIPAL_NAME}} via Telegram
8. Audit log entries written
9. Pipeline output saved to `data/action-items/` directory

Verify the Notion tasks were created:

**Remote:**
```
cd ${WORKSPACE} && node tools/notion-read.js tasks --search "test-transcript" --limit 10
```

Expected: Shows the draft tasks with their current status (approved or pending).

Verify the audit log was written:

**Remote:**
```
tail -10 ${WORKSPACE}/logs/action-items-audit.log
```

Expected: JSONL entries showing each decision with timestamps.

Clean up test tasks after verification:

**Remote:**
```
cd ${WORKSPACE} && node tools/notion-read.js tasks --search "Meridian" --json | python3 -c "
import json, sys
items = json.load(sys.stdin)
for item in items:
    if 'test' in item.get('context', '').lower() or 'Meridian' in item.get('description', ''):
        print(item.get('notion_id', 'unknown'))
"
```

For each test page ID, archive it:

**Remote:**
```
curl -s -X PATCH "https://api.notion.com/v1/pages/${TEST_PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_API_KEY}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"archived": true}'
```

If the full pipeline fails:
- **Extraction phase fails:** See step 6.4 troubleshooting.
- **Notion write fails:** Check write permissions. The integration bot must have "Insert content" capability enabled.
- **Telegram send fails:** Check bot token and chat ID. Verify the bot can message the client (client must have started a conversation with the bot first via `/start`).
- **Callback polling timeout:** This is expected if {{PRINCIPAL_NAME}} doesn't respond within 30 minutes during testing. The script should handle this gracefully and report timed-out items.
- **Webhook conflict:** The script must handle the webhook save/remove/restore pattern per `_deploy-common.md`. Check if the webhook was restored after the test.

#### 6.6 Clean Up Test Transcript `[AUTO]`

Remove the sample transcript to avoid confusion.

**Remote:**
```
rm -f ${WORKSPACE}/data/transcripts/test-transcript.txt
```

Expected: Test file removed.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| 401 on Claude API call | Invalid or expired Anthropic API key | Verify key starts with `sk-ant-api03-`. Regenerate at console.anthropic.com. Update `${WORKSPACE}/.env`. |
| Empty extraction results | Transcript too short or no actionable items | Check transcript is >50 chars and contains commitments, not just discussion. Review Claude prompt. |
| Fuzzy matching returns no results | People/Companies cache is empty or stale | Run `node tools/notion-sync.js --status` to check cache. Force refresh with `node tools/notion-sync.js --full`. |
| Fuzzy matching is too loose | Threshold too low, matching wrong people | Increase `matching.threshold` in `action-items.json` to 0.90 or higher. |
| Fuzzy matching misses valid names | Threshold too high or name normalization issue | Decrease threshold to 0.80. Check if names have prefixes/suffixes not being normalized. |
| Telegram message not received | Bot never started by client, or wrong chat ID | Client must send `/start` to the bot first. Verify `${TELEGRAM_CHAT_ID}` matches the client's user ID (check with `getChat`). |
| Inline keyboard buttons don't work | Callback data format mismatch | Verify the `callback_data` matches the pattern the polling loop expects. Check Telegram bot logs. |
| Approval times out | Client didn't see or respond to Telegram messages | Use `--resume` flag to re-send pending items. Check if Telegram notifications are enabled for the bot. |
| Notion task creation fails with 400 | Property names or types don't match database schema | Re-run database schema discovery (step 1.4). Update field mappings in scripts. |
| "Pending Approval" status rejected by Notion | Status value doesn't exist in the Tasks database | For `status` type properties, Notion auto-creates new values. For `select` type, verify the value exists or adapt to available options. |
| Webhook not restored after approval flow | Script crashed before cleanup | Manually restore: `curl -sS -X POST "https://api.telegram.org/bot${BOT_TOKEN}/setWebhook" -d "url=${SAVED_WEBHOOK}"`. The script's EXIT trap should handle this, but verify. |
| Pipeline hangs | Node.js subprocess not exiting | Check if `extract-actions.js` or `approval-flow.js` has an unresolved promise or open handle. Use `--verbose` for debug output. |
| Audit log not written | Permission denied or path doesn't exist | Check `${WORKSPACE}/logs/` directory exists and is writable. Check the `paths.auditLog` value in config. |
| Duplicate action items in Notion | Pipeline run twice on same transcript | The orchestrator should check for existing items by description before creating new ones. Add deduplication logic if missing. |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Create directories | Execute silently | Execute silently | Confirm before creating |
| 1.2 Store API keys | Pause for `[HUMAN_INPUT]` | Pause for `[HUMAN_INPUT]` | Pause for `[HUMAN_INPUT]` |
| 1.3 Read Notion DB IDs | Execute silently | Execute silently | Show results |
| 1.4 Discover status options | Execute silently | Show options, confirm mapping | Show options, confirm each |
| 1.5 Write action items config | Execute silently | Review config before writing | Review every field |
| 2.1 Install extract-actions.js | Execute silently | Confirm before writing | Review script, confirm |
| 3.1 Install approval-flow.js | Execute silently | Confirm before writing | Review script, confirm |
| 4.1 Install process-meeting.js | Execute silently | Confirm before writing | Review script, confirm |
| 5.1 Write TOOLS.md entries | Execute silently | Review entries before writing | Review each entry |
| 6.1 Validate config | Execute silently | Execute, show output | Show every check |
| 6.2 Validate Claude API | Execute silently | Execute, show result | Show request and response |
| 6.3 Validate Telegram bot | Execute silently | Execute, show result | Show request and response |
| 6.4 Test extraction | Execute, report result | Confirm before running, review output | Confirm, review prompt, review output |
| 6.5 Test full pipeline | Confirm before running | Confirm, alert client, monitor | Confirm each sub-step, monitor, review |
| 6.6 Clean up test files | Execute silently | Execute silently | Confirm before deleting |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed), `deploy-identity.md` (agent identity configured), `deploy-messaging-setup.md` (Telegram bot and messaging configured), `deploy-notion-workspace.md` (Notion config, People/Companies/Tasks databases mapped, `notion-read.js` and `notion-write.js` installed)
- **Enhanced by:** `deploy-fathom-pipeline.md` (Fathom transcripts can be auto-routed to this pipeline), `deploy-daily-briefing.md` (pending/overdue action items can feed into the daily briefing)
- **Required by:** None (standalone pipeline, but integrates with transcript sources)
- **Feeds into:** `deploy-daily-briefing.md` (approved action items appear in task summary), `deploy-notion-workspace.md` (tasks created in Notion Tasks database)
