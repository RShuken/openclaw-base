# Deploy Messaging Setup

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING
- **Messaging subcommands current as of 2026.4.15**: `openclaw channels` replaces the old per-provider subcommands. Use `openclaw channels list`, `openclaw channels send --channel <telegram|discord|slack|whatsapp> --to <target> --text "..."`, etc. Raw provider APIs (`curl https://api.telegram.org/bot<TOKEN>/...`) remain the always-works fallback and continue to be used in this skill where the native command doesn't cover the need (e.g., topic creation).
- **Config**: communication rules live in workspace files (`TOOLS.md`, `AGENTS.md`), not in a `messaging.*` config namespace. OpenClaw reads workspace files natively per `audit/_baseline.md` §5.
- **Model**: agent default (Codex-first); do not hardcode
- **ClawHub alternatives**: `agent-telegram`, `telegram-voice-group`, `rho-telegram-alerts` per §B — evaluate before scratch-building

## Purpose

Configure the communication backbone for the agent. Every other system (briefings, CRM, email, earnings, health tracking) delivers its output through these channels. This skill sets up organized messaging topics, routing rules, and communication style so notifications go to the right place and the signal-to-noise ratio stays high.

**When to use:** First-time setup, adding a new notification channel, or reorganizing an existing channel structure to prepare for new system deployments.

**What this skill does:**
1. Discovers existing messaging platforms, groups, topics, and bots
2. Reviews the deployment plan to determine which topics are needed
3. Maps existing or creates new topics for each planned system
4. Records the topic routing table to `${WORKSPACE}/TOOLS.md`
5. Configures secondary platform (Slack, WhatsApp, Discord) if applicable
6. Sets communication rules (message limits, cron visibility, file delivery)
7. Verifies messages route correctly to each configured topic

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${TELEGRAM_GROUP}` | Client profile or Telegram Bot API `getUpdates` response | `-1001234567890` |
| `${TELEGRAM_BOT_TOKEN}` | Client profile: messaging config | `7123456789:AAF...` |
| `${SLACK_USER_ID}` | Client profile: Slack user ID for the operator | `U0ABCDEF1` |
| `${TOPIC_ID_*}` | Created during Step 3 or from existing topic discovery | `1207` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if TOOLS.md already exists with a topic mapping:

**Remote:**
```
cat ${WORKSPACE}/TOOLS.md 2>/dev/null | head -40
```

Expected: Either the file contents (partially or fully configured) or an error indicating no file exists.

Check for an existing bot token in the OpenClaw config:

**Remote:**
```
cat ${OPENCLAW_CONFIG}/openclaw.json 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); tg=d.get('telegram',d.get('channels',{})); print(json.dumps(tg, indent=2))" 2>/dev/null || echo 'NO_TELEGRAM_CONFIG'
```

Expected: Telegram config section from openclaw.json, or no output if not yet configured.

Check the bot token directly (if stored in .env or config):

**Remote:**
```
grep -i 'telegram.*token\|bot.*token' ${OPENCLAW_CONFIG}/.env ${OPENCLAW_CONFIG}/openclaw.json ${WORKSPACE}/.env 2>/dev/null | head -5
```

Expected: Bot token value, or empty if not configured.

> **Note:** `openclaw config get channels`, `openclaw plugin list`, and all `openclaw telegram` subcommands do not exist in OpenClaw 2026.4.15+ (bumped 2026-04-20). Use raw Telegram Bot API calls and direct config file reads instead.

**Discovery questions to answer before proceeding:**
- Which messaging platforms does the client already have configured?
- Does the client have existing groups, topics, or channels? How many? What are they named?
- Which systems from the deployment plan will the client actually be installing? (This determines which topics are needed.)
- Does the client use a secondary platform (Slack, WhatsApp, Discord)?

## Adaptation Points

The recommended defaults below use OpenClaw's Clawd mascot as a starting point (Telegram primary with topics, Slack secondary in mention-only mode). Clients can use them as-is or customize any component — the 14-topic layout is a maximal example, most clients start with 3-5 and grow as they add systems.

| Component | Recommended Default | Customize When... |
|-----------|-------------------|-------------------|
| Primary platform | Telegram with topics in a single group | Client uses Discord (channels in a server), Slack (channels), or another platform as primary |
| Secondary platform | Slack in mention-only mode | Client does not use Slack, or uses WhatsApp/Discord as secondary instead |
| Topic structure | 14 dedicated topics (see table below) | Client has existing groups they want to reuse, wants fewer topics, or organizes differently |
| Topic-to-system mapping | 1:1 (each topic serves one system) | Client wants to consolidate topics (e.g., merge Earnings and Financials into one) |
| Communication rules | 2-message max per task, no cross-posting, failures only for cron | Client wants more verbose updates, different notification thresholds, or cross-posting for critical alerts |
| File delivery | Inline (send the actual file, not a link) | Link-based delivery if client has bandwidth constraints or prefers cloud links |
| Cron visibility | Failures only (successes are silent) | Client wants all cron runs logged, or wants cron notifications disabled entirely |
| Financial data channel | Locked down, DM-only topic | Standard topic if client has no sensitive financial data, or omit if not deploying cost/earnings tracking |

## Prerequisites

- OpenClaw agent installed and running (`openclaw/install.md`)
- Messaging bot token configured (Telegram bot token, Slack bot token, etc.)
- For Telegram: a group exists (or client is ready to create one) with the bot added as admin
- For Slack: a Slack app/bot exists with socket mode enabled
- A deployment plan (or at least a rough list of which systems will be deployed) so you know which topics to create

## What Gets Installed

### Recommended Topic Structure

This is the reference implementation's topic list. Each topic maps to one or more deploy skills. Present this full list to the client but only create topics for systems they plan to deploy. The client can map these to existing groups/topics or create new ones.

| # | Topic | Purpose | Used By | Create When... |
|---|-------|---------|---------|----------------|
| 1 | Daily Brief | Morning briefing delivery | `deploy-daily-briefing` | Deploying the daily briefing system |
| 2 | Personal CRM | Contact queries, follow-ups, nudges | `deploy-personal-crm` | Deploying the CRM |
| 3 | Email | Email triage, urgency alerts, draft proposals | `deploy-urgent-email` | Deploying email monitoring |
| 4 | Knowledge Base | KB ingestion confirmations, query results | `deploy-knowledge-base` | Deploying the knowledge base |
| 5 | Meta-Analysis | Advisory council digests, ranked recommendations | `deploy-advisory-council` | Deploying the advisory council |
| 6 | Video Ideas | Video pitch research, Asana card creation | `deploy-video-pipeline` | Deploying the video pipeline |
| 7 | Earnings | Earnings preview reports, company analysis | `deploy-earnings` | Deploying earnings tracking |
| 8 | Cron Updates | Automated job failures (successes are silent) | All cron-based systems | Always (any system with scheduled jobs) |
| 9 | Financials | Cost reports, model spending, sensitive financial data | `deploy-model-tracking`, `deploy-earnings` | Deploying cost tracking or earnings (DM-only recommended) |
| 10 | Health | Food logging, symptom tracking, weekly correlations | `deploy-food-journal` | Deploying health/food tracking |
| 11 | AI Tweets | Tweet curation, social media drafts | `deploy-social-tracking` | Deploying social tracking |
| 12 | Updates | General status updates, system announcements | Core system | Always (catch-all for general notifications) |
| 13 | Meeting Prep | Pre-meeting briefings with CRM context | `deploy-fathom-pipeline` | Deploying Fathom/meeting processing |
| 14 | Self-Improvement | Agent debugging, platform health digests, security council output | `deploy-platform-health`, `deploy-security-council` | Deploying health monitoring or security council |

**Note:** This is the recommended starting point, not a rigid requirement. Clients with existing channel structures should map their groups to these purposes rather than recreating everything from scratch.

### Communication Rules

These rules are recommended for all clients regardless of platform or topic structure. They keep the agent's messaging disciplined and prevent notification fatigue.

| Rule | Description |
|------|-------------|
| **Two-message max** | Per task, the agent sends at most two messages: an acknowledgment, then the result. No play-by-play narration of intermediate steps. |
| **No cross-posting** | Each topic receives only its designated content type. A CRM update goes to Personal CRM, not also to Updates. |
| **Files inline** | When sending files, send the actual file as an attachment, not a link to it. |
| **Failures surface, successes are silent** | Cron jobs and automated tasks only notify on failure. Success is the default and does not need a message. |
| **Complete replies** | Every response is one complete message. No "thinking..." or "working on it..." intermediate messages. |
| **No unsolicited broadcasts** | The agent does not message the operator unless it has something specific to deliver (a scheduled briefing, an alert, a requested result). |

### Slack Configuration (If Applicable)

The recommended Slack setup uses mention-only mode with a single-user allowlist. This prevents the agent from responding to every message in every channel.

| Setting | Recommended Default | Purpose |
|---------|-------------------|---------|
| Connection mode | Socket mode | Real-time, no public endpoint needed |
| Response trigger | Mention-only (`@agent`) | Agent only responds when directly mentioned |
| User allowlist | Operator only (single user) | Prevents other team members from invoking the agent |
| Auto-reaction | Eyes emoji on receipt | Visual confirmation the agent saw the message |
| Message style | Single complete message | No intermediate "thinking..." messages |

## Steps

### Phase 1: Discovery

#### 1.1 Discover Existing Messaging Setup `[AUTO]`

Before creating anything, audit what the client already has. This is the most important step. Many clients already have groups, channels, or bots configured.

> **Note:** All `openclaw telegram`, `openclaw config get channels`, and `openclaw plugin list` commands do not exist in OpenClaw 2026.4.15+ (bumped 2026-04-20). Use raw Telegram Bot API calls instead.

Check if a Telegram bot token is available:

**Remote:**
```
grep -i 'telegram\|bot_token\|BOT_TOKEN' ${OPENCLAW_CONFIG}/.env ${OPENCLAW_CONFIG}/openclaw.json ${WORKSPACE}/.env 2>/dev/null
```

Expected: Bot token value. Record it as `${TELEGRAM_BOT_TOKEN}`.

If no bot token is found, ask the client. They need to create a Telegram bot via @BotFather and provide the token.

#### 1.2 Discover Existing Telegram Groups and Topics `[AUTO]`

Use the Telegram Bot API directly to discover groups the bot is in.

**Operator** (run from your terminal, not the client's machine):
```bash
curl -sS "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getUpdates" | python3 -c "
import json, sys
data = json.load(sys.stdin)
chats = {}
for r in data.get('result', []):
    for key in ['message', 'my_chat_member', 'channel_post']:
        msg = r.get(key, {})
        chat = msg.get('chat', {})
        if chat.get('id'):
            chats[chat['id']] = {'title': chat.get('title',''), 'type': chat.get('type','')}
for cid, info in chats.items():
    print(f'{cid}\t{info[\"type\"]}\t{info[\"title\"]}')
"
```

Expected: List of chat IDs, types, and titles. Supergroups with forum enabled are the target. Record the primary group ID as `${TELEGRAM_GROUP}`.

If no groups appear: The bot has not received any messages yet. Have the client add the bot to their group, send a message, then retry.

To check if topics (forum mode) are enabled, and list existing topics:

**Operator:**
```bash
curl -sS "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getForumTopicIconStickers" | python3 -c "import json,sys; print('Forum mode available' if json.load(sys.stdin).get('ok') else 'Forum mode not available')"
```

> **Note:** There is no Bot API method to list all existing forum topics. The bot only sees topics it has received messages in (via `getUpdates`). If the group already has topics, have the client send a message in each topic so the bot can discover them, or ask the client for the topic thread IDs.

#### 1.3 Discover Existing Slack Configuration `[AUTO]`

Only run this if the client uses Slack as a secondary platform.

> **Note:** `openclaw slack list-channels` does not exist. Check for Slack configuration in the OpenClaw config file directly.

**Remote:**
```
grep -i 'slack' ${OPENCLAW_CONFIG}/openclaw.json 2>/dev/null || echo 'NO_SLACK_CONFIG'
```

Expected: Slack configuration, or empty if not configured.

#### 1.4 Check for Existing Topic Mapping `[AUTO]`

Check if a previous deployment already created the topic routing file.

**Remote:**
```
cat ${WORKSPACE}/TOOLS.md 2>/dev/null
```

Expected: Either the full TOOLS.md content (already deployed) or an error/empty (first time).

If already exists: Read the content carefully. If it has correct topic IDs and matches the client's current setup, skip to Phase 3 (Communication Rules). If it exists but is outdated or incomplete, proceed with updates.

**Record what you find:**

| Question | Answer |
|----------|--------|
| Primary platform | _____________ |
| Secondary platform (if any) | _____________ |
| Existing groups/topics count | _____________ |
| Existing group names | _____________ |
| Bot already added as admin? | _____________ |
| TOOLS.md already has topic mapping? | _____________ |

### Phase 2: Topic Planning and Creation

#### 2.1 Review the Deployment Plan `[HUMAN_INPUT]`

The topic list is driven by which systems the client will deploy. Before choosing topics, confirm the deployment plan.

Ask the client:
> "Here are the 14 recommended topics and which system each one serves. Let me know which systems you plan to deploy so we only create the topics you need. You can always add more later."

Present the recommended topic table from "What Gets Installed" above. Mark each topic as **needed**, **skip**, or **later** based on the client's deployment plan.

**Core topics** (recommended for all deployments):
- Cron Updates -- needed as soon as any cron-based system is deployed
- Updates -- general catch-all, useful from day one

**System-specific topics** -- only create when deploying the matching system:
- Daily Brief, Personal CRM, Email, Knowledge Base, Meta-Analysis, Video Ideas, Earnings, Financials, Health, AI Tweets, Meeting Prep, Self-Improvement

Expected: A clear list of which topics to create, which to map to existing channels, and which to skip.

#### 2.2 Map or Create Topics `[GUIDED]`

For each needed topic, the client either maps it to an existing group/topic or creates a new one.

**Walk the client through this worksheet:**

| # | Recommended Topic | Client's Mapping | Action | Group/Topic ID |
|---|-------------------|-----------------|--------|----------------|
| 1 | Daily Brief | _____________ | create / map existing / skip | _____________ |
| 2 | Personal CRM | _____________ | create / map existing / skip | _____________ |
| 3 | Email | _____________ | create / map existing / skip | _____________ |
| 4 | Knowledge Base | _____________ | create / map existing / skip | _____________ |
| 5 | Meta-Analysis | _____________ | create / map existing / skip | _____________ |
| 6 | Video Ideas | _____________ | create / map existing / skip | _____________ |
| 7 | Earnings | _____________ | create / map existing / skip | _____________ |
| 8 | Cron Updates | _____________ | create / map existing / skip | _____________ |
| 9 | Financials | _____________ | create / map existing / skip | _____________ |
| 10 | Health | _____________ | create / map existing / skip | _____________ |
| 11 | AI Tweets | _____________ | create / map existing / skip | _____________ |
| 12 | Updates | _____________ | create / map existing / skip | _____________ |
| 13 | Meeting Prep | _____________ | create / map existing / skip | _____________ |
| 14 | Self-Improvement | _____________ | create / map existing / skip | _____________ |
| -- | *(client custom)* | _____________ | create | _____________ |

**For topics marked "create":**

Create a new Telegram topic in the primary group using the Telegram Bot API directly.

**Operator** (run from your terminal, repeat for each topic):
```bash
curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/createForumTopic" \
  -H "Content-Type: application/json" \
  -d '{"chat_id": "${TELEGRAM_GROUP}", "name": "<topic-name>"}' \
  | python3 -c "import json,sys; r=json.load(sys.stdin); print(f'Thread ID: {r[\"result\"][\"message_thread_id\"]}' if r.get('ok') else f'ERROR: {r}')"
```

> **Important:** Replace `${TELEGRAM_GROUP}` with the actual group ID and `<topic-name>` with the topic name (e.g., `Daily Brief`) before running. Repeat for each topic to create. Record each returned thread ID in the worksheet.

**Optional:** Set a topic icon color with the `icon_color` parameter (integer). Available colors: `7322096` (blue), `16766590` (yellow), `13338331` (violet), `9367192` (green), `16749490` (rose), `16478047` (red).

Expected: JSON response with `{"ok": true, "result": {"message_thread_id": 123, ...}}`. Record the `message_thread_id` value.

If this fails:
- **"Bad Request: not enough rights"**: The bot lacks admin privileges. Promote the bot to admin in the Telegram group settings (must have "Manage Topics" permission) and retry.
- **"Bad Request: TOPIC_NOT_MODIFIED"**: A topic with that name may already exist.
- **"Bad Request: CHAT_NOT_MODIFIED"**: Forum mode may not be enabled on the group. Have the client enable Topics in group settings.

If already exists: If discovery in Phase 1 found a topic with the same name, use its existing thread ID instead of creating a duplicate.

**For topics marked "map existing":**

Record the existing group or topic ID. No creation needed. If the client uses separate groups instead of topics within one group, record the group chat ID instead.

**For topics marked "skip":**

No action. These can be added later when the corresponding system is deployed.

**Client has custom topics not in the list?**

Add them. The recommended 14 are a starting point. If the client has a topic for something we do not cover (e.g., a project-specific channel), record it so other systems can be configured to avoid posting there.

#### 2.3 Record the Topic Mapping `[GUIDED]`

Write the final mapping to `${WORKSPACE}/TOOLS.md` so every other deploy skill can look up where to send its messages.

Create the file `${WORKSPACE}/TOOLS.md` with the topic routing table. The file should contain the following sections with actual values substituted throughout:

**File structure for `${WORKSPACE}/TOOLS.md`:**

```
# Messaging Configuration

## Platform
- **Primary:** [Telegram / Discord / other]
- **Secondary:** [Slack / WhatsApp / none]
- **Group ID:** [actual group ID]

## Topic Routing

| Topic | Thread/Channel ID | Platform | Notes |
|-------|-------------------|----------|-------|
| Daily Brief | [actual ID] | Telegram | |
| Personal CRM | [actual ID] | Telegram | |
| Email | [actual ID] | Telegram | |
| Knowledge Base | [actual ID] | Telegram | |
| Meta-Analysis | [actual ID] | Telegram | |
| Video Ideas | [actual ID] | Telegram | |
| Earnings | [actual ID] | Telegram | |
| Cron Updates | [actual ID] | Telegram | Failures only |
| Financials | [actual ID] | Telegram | DM-only / locked |
| Health | [actual ID] | Telegram | |
| AI Tweets | [actual ID] | Telegram | |
| Updates | [actual ID] | Telegram | |
| Meeting Prep | [actual ID] | Telegram | |
| Self-Improvement | [actual ID] | Telegram | |

## Communication Rules
- Max messages per task: 2
- Cross-posting: disabled
- File delivery: inline
- Cron notifications: failures only
- Narration: disabled
```

If the client also uses Slack, add a Slack section:

```
## Slack
- Mode: mention-only
- Allowlist: [actual Slack user ID]
- Auto-reaction: eyes
- Message style: single-complete
```

**How to write this file remotely:**

The operator must construct the complete file content with all actual values (group IDs, topic thread IDs, platform names) substituted in, then send it as a write command. Use an unquoted heredoc (without single quotes around the delimiter) so the shell on the client machine receives the literal text.

**Remote:**
```
cat > ${WORKSPACE}/TOOLS.md << TOOLSEOF
[full file content with actual values]
TOOLSEOF
```

> **Critical (T10 fix):** Do NOT use a single-quoted heredoc (`<< 'TOOLSEOF'`). While single-quoted heredocs prevent unwanted shell expansion on the client, that is irrelevant here because the operator has already substituted all values before sending the command through the API. The command string sent through the session API is the final text. If the file content contains any `$` characters that should appear literally (e.g., in script examples), escape them as `\$`.

Remove rows from the topic table for topics that were skipped. Add rows for any custom topics the client requested.

Expected: File exists at `${WORKSPACE}/TOOLS.md` with all actual topic/channel IDs filled in, no placeholder text.

If this fails: Check that the `${WORKSPACE}` directory exists. Create it first if missing.

If already exists: Compare with the new content. If the existing file has different topic IDs or is missing entries, back up as `TOOLS.md.bak` and write the updated version.

### Phase 3: Platform Configuration

#### 3.1 Configure the Secondary Platform `[GUIDED]`

Not all clients use a secondary platform. This step is conditional -- skip if the client only uses one platform.

> **Note:** `openclaw config set slack.*` commands may not work in OpenClaw 2026.4.15+ (bumped 2026-04-20) (the `slack` config namespace needs verification). If config set fails, document the Slack configuration in TOOLS.md instead and configure Slack through its own settings.

**If the client uses Slack (recommended default):**

The recommended Slack setup uses mention-only mode. Since `openclaw config set slack.*` may not exist, configure via the Slack app settings and document in TOOLS.md:

1. **Slack App Settings** (done by client in browser): Enable Socket Mode in the app settings, configure event subscriptions for `app_mention` only.
2. **Document in TOOLS.md** (done during Step 2.3): Add a Slack section recording the configuration.

If `openclaw config set` works for slack keys, use these commands:

**Remote:**
```
openclaw config set slack.mode socket 2>&1
openclaw config set slack.trigger mention-only 2>&1
openclaw config set slack.allowlist '${SLACK_USER_ID}' 2>&1
```

If any command returns a schema validation error, STOP trying config set for Slack. Document the configuration in TOOLS.md instead.

**If the client uses Discord:**

Discord channels map naturally to Telegram topics. Use channel IDs in the topic routing table with `Platform: Discord` instead of `Platform: Telegram`.

**If no secondary platform:**

Skip this step. Note in the client profile that only the primary platform is configured.

#### 3.2 Set Communication Rules `[AUTO]`

Configure the agent's messaging behavior. These rules are recommended for all clients but can be adjusted per the Adaptation Points.

> **Note:** `openclaw config set messaging.*` does NOT work in OpenClaw 2026.4.15+ (bumped 2026-04-20). The `messaging` config namespace does not exist. Communication rules are documented in TOOLS.md (written in Step 2.3) and AGENTS.md instead. OpenClaw reads these workspace files as system context for every session.

The communication rules should already be included in the TOOLS.md file written in Step 2.3 (see the "Communication Rules" section of the file template). If they were not included, append them now:

**Remote:**
```
cat >> ${WORKSPACE}/AGENTS.md << 'EOF'

## Communication Rules

- Max messages per task: 2 (acknowledgment + result). No play-by-play narration.
- No cross-posting: each topic receives only its designated content type.
- File delivery: inline (send the actual file, not a link).
- Cron notifications: failures only. Success is silent.
- Complete replies: every response is one complete message. No "thinking..." intermediates.
- No unsolicited broadcasts: only message when delivering something specific.
EOF
```

Expected: AGENTS.md contains a "Communication Rules" section. Verify:

**Remote:**
```
grep -c 'Communication Rules' ${WORKSPACE}/AGENTS.md
```

If already exists: Check if the section is already present. If so, skip to avoid duplicates.

**Customization options (present to client if they want changes):**

| Rule | Default | Alternatives |
|------|---------|-------------|
| Max messages per task | 2 (ack + result) | 1 (result only, no ack), 3 (ack + progress + result) |
| Narration | Disabled | Enabled for debugging, then disable |
| File delivery | Inline (actual file) | Link (for bandwidth-constrained setups) |
| Cross-posting | Disabled | Enabled for critical alerts only (e.g., security issues go to both Cron Updates and the operator's DM) |
| Cron notifications | Failures only | All runs (verbose), disabled (fully silent) |

### Phase 4: Verification

#### 4.1 Quick Topic Test `[AUTO]`

Send test messages to a few key topics to confirm routing works. Use the Telegram Bot API directly.

**Operator** (run from your terminal):
```bash
# Test the Updates topic (replace TOPIC_ID_UPDATES with the actual thread ID)
curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d '{"chat_id": "${TELEGRAM_GROUP}", "message_thread_id": ${TOPIC_ID_UPDATES}, "text": "Messaging setup test: Updates topic is working."}' \
  | python3 -c "import json,sys; r=json.load(sys.stdin); print('OK' if r.get('ok') else f'ERROR: {r}')"
```

```bash
# Test the Cron Updates topic (replace TOPIC_ID_CRON with the actual thread ID)
curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d '{"chat_id": "${TELEGRAM_GROUP}", "message_thread_id": ${TOPIC_ID_CRON}, "text": "Messaging setup test: Cron Updates topic is working."}' \
  | python3 -c "import json,sys; r=json.load(sys.stdin); print('OK' if r.get('ok') else f'ERROR: {r}')"
```

> **Important:** Replace `${TELEGRAM_GROUP}`, `${TELEGRAM_BOT_TOKEN}`, and the topic thread IDs with actual values. Use `parse_mode: "HTML"` for formatted messages (not Markdown -- AI-generated content breaks the strict Markdown parser).

Expected: Messages appear in the correct Telegram topics. The client should confirm they see them.

If this fails: Check that the group ID and thread IDs are correct. Common errors:
- **"Bad Request: message thread not found"**: The thread ID is wrong or the topic was deleted.
- **"Bad Request: chat not found"**: The group ID is wrong or the bot is not a member.

#### 4.2 Full Topic Test `[GUIDED]`

Send a test message to every configured topic. The operator must loop through all topic IDs from TOOLS.md.

**Operator** (adapt for each configured topic):
```bash
# For each topic in TOOLS.md, send a test message
for topic_name_and_id in "Daily Brief:${TOPIC_ID_1}" "Personal CRM:${TOPIC_ID_2}" "Cron Updates:${TOPIC_ID_8}"; do
  name="${topic_name_and_id%%:*}"
  tid="${topic_name_and_id##*:}"
  result=$(curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
    -H "Content-Type: application/json" \
    -d "{\"chat_id\": \"${TELEGRAM_GROUP}\", \"message_thread_id\": ${tid}, \"text\": \"Test: ${name} topic is working.\"}" \
    | python3 -c "import json,sys; r=json.load(sys.stdin); print('OK' if r.get('ok') else 'FAIL')")
  echo "${name} (thread ${tid}): ${result}"
done
```

Expected: Each topic reports `OK`. The client should see test messages in every configured topic.

#### 4.3 Slack Verification `[AUTO]`

Only run if Slack was configured in Phase 3.

> **Note:** `openclaw slack status` does not exist in OpenClaw 2026.4.15+ (bumped 2026-04-20). Verify Slack by testing mention-based interaction manually.

Test by mentioning the bot in a Slack channel and confirming it responds. If the bot does not respond, check the Slack app settings: Socket Mode must be enabled, event subscriptions must include `app_mention`, and the bot must be invited to the channel.

#### 4.4 Communication Rules Verification `[AUTO]`

Confirm communication rules are documented in AGENTS.md and TOOLS.md.

> **Note:** `openclaw config show messaging` does not exist. Communication rules are documented in workspace files, not config keys.

**Remote:**
```
grep -c 'Communication Rules' ${WORKSPACE}/AGENTS.md 2>/dev/null || echo 'NOT FOUND in AGENTS.md'
grep -c 'Communication Rules' ${WORKSPACE}/TOOLS.md 2>/dev/null || echo 'NOT FOUND in TOOLS.md'
```

Expected: At least one file contains a "Communication Rules" section.

#### 4.5 TOOLS.md Verification `[AUTO]`

Confirm the topic routing file is complete and accurate.

**Remote:**
```
cat ${WORKSPACE}/TOOLS.md
```

Expected: File contains a complete topic routing table with actual thread/channel IDs (not placeholder text like `<id>` or `${VARIABLE}`), platform labels, and communication rules.

If this fails: If the file is missing or has placeholders, return to Step 2.3 and rewrite it with actual values.

### Phase 5: Post-Deployment

#### 5.1 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:

- Which messaging platforms were configured (primary and secondary)
- How many topics were created vs. mapped to existing groups
- Which topics were skipped and why (not deploying that system yet)
- Any custom topics the client added
- Communication rule overrides (if any differ from defaults)
- Note: "Messaging setup deployed on [date]"

## Verification

After deployment, the messaging setup is confirmed working when:

1. **TOOLS.md exists** at `${WORKSPACE}/TOOLS.md` with a complete topic routing table containing actual IDs
2. **Each configured topic receives test messages** in the correct group/thread
3. **Secondary platform (if any) connects and responds** to mentions correctly
4. **Communication rules are set** in the agent config
5. **No cross-posting** -- a test message sent to one topic does not appear in another

**Remote:**
```
test -f ${WORKSPACE}/TOOLS.md && echo 'TOOLS.md exists' || echo 'TOOLS.md MISSING'
```

**Remote:**
```
grep -c 'Communication Rules' ${WORKSPACE}/AGENTS.md ${WORKSPACE}/TOOLS.md 2>/dev/null
```

> **Note:** `openclaw config show messaging` and `openclaw telegram test-all-topics` do not exist. Verify communication rules via workspace files and test topics via the Telegram Bot API (see Phase 4 above).

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Bot cannot create topics | Bot lacks admin privileges in the Telegram group | Promote the bot to admin in group settings |
| Messages go to wrong topic | Topic/thread IDs are wrong in TOOLS.md | Verify thread IDs by sending test messages via the Bot API. Update TOOLS.md with correct IDs. |
| Existing groups not discovered | Bot has not received messages in all groups | Bot must be a member of each group and receive at least one message to appear in `getUpdates`. Add the bot, send a message, then re-run discovery. |
| TOOLS.md contains `${VARIABLE}` text | Operator sent the write command without substituting values | Rewrite TOOLS.md with actual literal values, not variable placeholders |
| Slack does not respond to mentions | Socket mode not connected or bot token invalid | Check Slack app settings: Socket Mode must be enabled, bot must have `app_mention` event subscription, bot must be in the channel |
| Slack responds to all messages | Event subscription too broad | Restrict Slack event subscriptions to `app_mention` only (in Slack app settings, not via OpenClaw config) |
| File sent as link instead of attachment | Communication rules not documented | Update AGENTS.md communication rules section to specify inline file delivery |
| Cron successes flooding the channel | Communication rules not enforced | Update AGENTS.md to specify "failures only" for cron notifications |
| Topic not found error | Topic was deleted, renamed, or thread ID changed | Re-create the topic via the Bot API (`createForumTopic`) and update the thread ID in TOOLS.md |
| Client has more groups than the 14 defaults | Client has custom topics for their workflow | Record them in TOOLS.md so other systems avoid posting there |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| Phase 1: Discovery (1.1-1.4) | Execute all silently | Execute all silently | Confirm each command |
| Phase 2.1: Review Deployment Plan | Always asks client (HUMAN_INPUT) | Always asks client | Always asks client |
| Phase 2.2: Map or Create Topics | Execute | Confirm before each create | Confirm each command |
| Phase 2.3: Record Topic Mapping | Execute | Confirm before writing | Show content, confirm |
| Phase 3.1: Secondary Platform | Execute | Confirm before configuring | Confirm each command |
| Phase 3.2: Communication Rules | Execute | Execute | Confirm each rule |
| Phase 4: Verification | Execute all | Execute all | Confirm each test |
| Phase 5: Update Client Profile | Execute | Execute | Show changes, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed and running), `deploy-identity.md` (personality and tone affect how messages are written -- deploy identity first so the agent's voice is set before it starts sending messages)
- **Required by:** Every system that sends notifications, including:
  - `deploy-daily-briefing.md` (Daily Brief topic)
  - `deploy-personal-crm.md` (Personal CRM topic)
  - `deploy-urgent-email.md` (Email topic)
  - `deploy-knowledge-base.md` (Knowledge Base topic)
  - `deploy-advisory-council.md` (Meta-Analysis topic)
  - `deploy-video-pipeline.md` (Video Ideas topic)
  - `deploy-earnings.md` (Earnings topic)
  - `deploy-food-journal.md` (Health topic)
  - `deploy-social-tracking.md` (AI Tweets topic)
  - `deploy-fathom-pipeline.md` (Meeting Prep topic)
  - `deploy-platform-health.md` (Self-Improvement topic)
  - `deploy-security-council.md` (Self-Improvement topic)
  - `deploy-model-tracking.md` (Financials topic)
  - `deploy-health-monitoring.md` (Cron Updates topic)
  - `deploy-db-backups.md` (Cron Updates topic)
  - `deploy-git-autosync.md` (Cron Updates topic)
- **Related:** `deploy-identity.md` (message tone), `deploy-prompt-guide.md` (agent's communication patterns align with prompting style)
