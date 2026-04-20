# Deploy Messaging Setup

## Purpose

Configure the communication backbone for the agent. Every other system (briefings, CRM, email, earnings, health tracking) delivers its output through these channels. This skill sets up organized messaging topics, routing rules, and communication style so notifications go to the right place and the signal-to-noise ratio stays high.

**When to use:** First-time setup, adding a new notification channel, or reorganizing an existing channel structure to prepare for new system deployments.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "cat ${WORKSPACE}/TOOLS.md 2>/dev/null | head -40"
cmd --session <id> --command "openclaw config get channels"
cmd --session <id> --command "openclaw plugin list | grep -iE 'telegram|slack|discord|whatsapp'"
```
- Which messaging platforms does the client already have configured?
- Does the client have existing groups, topics, or channels? How many? What are they named?
- Which systems from the deployment plan will the client actually be installing? (This determines which topics are needed.)
- Does the client use a secondary platform (Slack, WhatsApp, Discord)?

## Adaptation Points

The recommended defaults below come from the reference implementation (Matt Berman's "Clawd" agent, running 26 systems with 14 Telegram topics and Slack as secondary). Clients can use them as-is or customize any component.

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

- OpenClaw agent installed and running
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

### 1. Discover Existing Messaging Setup

Before creating anything, audit what the client already has. This is the most important step. Many clients already have groups, channels, or bots configured.

**Discover platforms:**
```
cmd --session <id> --command "openclaw config get channels"
cmd --session <id> --command "openclaw plugin list | grep -iE 'telegram|slack|discord|whatsapp'"
```

**Discover existing Telegram groups/topics:**
```
cmd --session <id> --command "openclaw telegram list-groups"
cmd --session <id> --command "openclaw telegram list-topics --group-id ${TELEGRAM_GROUP}"
```

**Discover existing Slack channels (if applicable):**
```
cmd --session <id> --command "openclaw slack list-channels 2>/dev/null"
```

**Check for existing topic mapping:**
```
cmd --session <id> --command "cat ${WORKSPACE}/TOOLS.md 2>/dev/null"
```

Record what you find:

| Question | Answer |
|----------|--------|
| Primary platform | _____________ |
| Secondary platform (if any) | _____________ |
| Existing groups/topics count | _____________ |
| Existing group names | _____________ |
| Bot already added as admin? | _____________ |
| TOOLS.md already has topic mapping? | _____________ |

### 2. Review the Deployment Plan

The topic list is driven by which systems the client will deploy. Before choosing topics, confirm the deployment plan.

Ask the client:
> "Here are the 14 recommended topics and which system each one serves. Let me know which systems you plan to deploy so we only create the topics you need. You can always add more later."

Present the recommended topic table from above. Mark each topic as **needed**, **skip**, or **later** based on the client's deployment plan.

**Core topics** (recommended for all deployments):
- Cron Updates -- needed as soon as any cron-based system is deployed
- Updates -- general catch-all, useful from day one

**System-specific topics** -- only create when deploying the matching system:
- Daily Brief, Personal CRM, Email, Knowledge Base, Meta-Analysis, Video Ideas, Earnings, Financials, Health, AI Tweets, Meeting Prep, Self-Improvement

### 3. Map or Create Topics

This is where discovery pays off. For each needed topic, the client either maps it to an existing group/topic or creates a new one.

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
```
cmd --session <id> --command "openclaw telegram create-topic --group-id ${TELEGRAM_GROUP} --name '<topic-name>'"
```

**For topics marked "map existing":**
Record the existing group or topic ID. No creation needed. If the client uses separate groups instead of topics within one group, record the group chat ID instead.

**For topics marked "skip":**
No action. These can be added later when the corresponding system is deployed.

**Client has custom topics not in the list?**
Add them. The recommended 14 are a starting point. If the client has a topic for something we do not cover (e.g., a project-specific channel), record it so other systems can be configured to avoid posting there.

### 4. Record the Topic Mapping

Write the final mapping to `${WORKSPACE}/TOOLS.md` so every other deploy skill can look up where to send its messages.

**Recommended format:**

```
cmd --session <id> --command "cat > ${WORKSPACE}/TOOLS.md << 'TOOLSEOF'
# Messaging Configuration

## Platform
- **Primary:** [Telegram / Discord / other]
- **Secondary:** [Slack / WhatsApp / none]
- **Group ID:** [group-id]

## Topic Routing

| Topic | Thread/Channel ID | Platform | Notes |
|-------|-------------------|----------|-------|
| Daily Brief | <id> | Telegram | |
| Personal CRM | <id> | Telegram | |
| Email | <id> | Telegram | |
| Knowledge Base | <id> | Telegram | |
| Meta-Analysis | <id> | Telegram | |
| Video Ideas | <id> | Telegram | |
| Earnings | <id> | Telegram | |
| Cron Updates | <id> | Telegram | Failures only |
| Financials | <id> | Telegram | DM-only / locked |
| Health | <id> | Telegram | |
| AI Tweets | <id> | Telegram | |
| Updates | <id> | Telegram | |
| Meeting Prep | <id> | Telegram | |
| Self-Improvement | <id> | Telegram | |

## Communication Rules
- Max messages per task: 2
- Cross-posting: disabled
- File delivery: inline
- Cron notifications: failures only
- Narration: disabled
TOOLSEOF"
```

Replace `<id>` values with actual thread/channel IDs from Step 3. Remove rows for topics that were skipped.

**Reference: Matt's TOOLS.md** (use as-is if client keeps all 14 defaults in a single Telegram group):

```
cmd --session <id> --command "cat > ${WORKSPACE}/TOOLS.md << 'TOOLSEOF'
# Messaging Configuration

## Platform
- **Primary:** Telegram
- **Secondary:** Slack (mention-only)
- **Group ID:** ${TELEGRAM_GROUP}

## Topic Routing

| Topic | Thread ID | Platform | Notes |
|-------|-----------|----------|-------|
| Daily Brief | <id> | Telegram | |
| Personal CRM | <id> | Telegram | |
| Email | <id> | Telegram | |
| Knowledge Base | <id> | Telegram | |
| Meta-Analysis | <id> | Telegram | |
| Video Ideas | <id> | Telegram | |
| Earnings | <id> | Telegram | |
| Cron Updates | <id> | Telegram | Failures only |
| Financials | <id> | Telegram | DM-only |
| Health | <id> | Telegram | |
| AI Tweets | <id> | Telegram | |
| Updates | <id> | Telegram | |
| Meeting Prep | <id> | Telegram | |
| Self-Improvement | <id> | Telegram | |

## Slack
- Mode: mention-only
- Allowlist: <operator-slack-user-id>
- Auto-reaction: eyes
- Message style: single-complete

## Communication Rules
- Max messages per task: 2
- Cross-posting: disabled
- File delivery: inline
- Cron notifications: failures only
- Narration: disabled
TOOLSEOF"
```

### 5. Configure the Secondary Platform (If Applicable)

Not all clients use a secondary platform. This step is conditional.

**If the client uses Slack (recommended default):**

```
cmd --session <id> --command "openclaw config set slack.mode socket"
cmd --session <id> --command "openclaw config set slack.trigger mention-only"
cmd --session <id> --command "openclaw config set slack.allowlist '<operator-slack-user-id>'"
cmd --session <id> --command "openclaw config set slack.auto-reaction eyes"
cmd --session <id> --command "openclaw config set slack.message-style single-complete"
```

Verify:
```
cmd --session <id> --command "openclaw slack test"
```

**If the client uses WhatsApp:**

WhatsApp does not support topics or channels, so routing works differently. The agent sends all messages to a single WhatsApp chat (or uses separate contacts/groups for separation). Record the WhatsApp configuration in TOOLS.md:

```
## WhatsApp
- Chat ID: <whatsapp-chat-id>
- Mode: [all-in-one / separate-groups]
- Notes: [any client-specific WhatsApp configuration]
```

**If the client uses Discord:**

Discord channels map naturally to Telegram topics. Use channel IDs in the topic routing table with `Platform: Discord` instead of `Platform: Telegram`.

**If no secondary platform:**

Skip this step. Note in the client profile that only the primary platform is configured.

### 6. Set Communication Rules

Configure the agent's messaging behavior. These rules are recommended for all clients but can be adjusted.

```
cmd --session <id> --command "openclaw config set messaging.max-messages-per-task 2"
cmd --session <id> --command "openclaw config set messaging.narration false"
cmd --session <id> --command "openclaw config set messaging.file-delivery inline"
cmd --session <id> --command "openclaw config set messaging.cross-posting false"
cmd --session <id> --command "openclaw config set messaging.cron-updates failures-only"
```

**Customization options:**

| Rule | Default | Alternatives |
|------|---------|-------------|
| Max messages per task | 2 (ack + result) | 1 (result only, no ack), 3 (ack + progress + result) |
| Narration | Disabled | Enabled for debugging, then disable |
| File delivery | Inline (actual file) | Link (for bandwidth-constrained setups) |
| Cross-posting | Disabled | Enabled for critical alerts only (e.g., security issues go to both Cron Updates and the operator's DM) |
| Cron notifications | Failures only | All runs (verbose), disabled (fully silent) |

### 7. Verify

Test that messages route correctly to each configured topic.

**Quick test (send to a few key topics):**
```
cmd --session <id> --command "openclaw telegram send --topic 'Updates' --message 'Messaging setup test: Updates topic is working.'"
cmd --session <id> --command "openclaw telegram send --topic 'Cron Updates' --message 'Messaging setup test: Cron Updates topic is working.'"
```

**Full test (all configured topics):**
```
cmd --session <id> --command "openclaw telegram test-all-topics"
```

**Slack verification (if configured):**
```
cmd --session <id> --command "openclaw slack status"
```

Expected Slack output:
- Connection: Socket mode, connected
- Trigger: Mention-only
- Allowlist: 1 user configured
- Auto-reaction: enabled

Test Slack by mentioning the bot and confirming it reacts with eyes, then responds with a single complete message.

**Communication rules verification:**
```
cmd --session <id> --command "openclaw config show messaging"
```

**TOOLS.md verification:**
```
cmd --session <id> --command "cat ${WORKSPACE}/TOOLS.md"
```

Confirm all topic IDs are recorded and the routing table matches what was created or mapped in Step 3.

### 8. Update Client Profile

After deployment, update `clients/<name>.md` with:

- Which messaging platforms were configured (primary and secondary)
- How many topics were created vs. mapped to existing groups
- Which topics were skipped and why (not deploying that system yet)
- Any custom topics the client added
- Communication rule overrides (if any differ from defaults)
- Note: "Messaging setup deployed on [date]"

## Verification

After deployment, the messaging setup is confirmed working when:

1. **TOOLS.md exists** at `${WORKSPACE}/TOOLS.md` with a complete topic routing table
2. **Each configured topic receives test messages** in the correct group/thread
3. **Secondary platform (if any) connects and responds** to mentions correctly
4. **Communication rules are set** in the agent config
5. **No cross-posting** -- a test message sent to one topic does not appear in another

```
cmd --session <id> --command "test -f ${WORKSPACE}/TOOLS.md && echo 'TOOLS.md exists' || echo 'TOOLS.md MISSING'"
cmd --session <id> --command "openclaw config show messaging"
cmd --session <id> --command "openclaw telegram test-all-topics"
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Bot cannot create topics | Bot lacks admin privileges in the Telegram group | Promote the bot to admin in group settings |
| Messages go to wrong topic | Topic/thread IDs are wrong in TOOLS.md | Run `openclaw telegram list-topics` and update the IDs |
| Existing groups not discovered | `list-groups` does not show all groups | Bot must be a member of each group to see it. Add the bot, then re-run discovery. |
| Slack does not respond to mentions | Socket mode not connected or bot token invalid | Check `openclaw slack status`, restart the connection, verify the bot token |
| Slack responds to all messages | Trigger mode not set to mention-only | Run `openclaw config set slack.trigger mention-only` |
| File sent as link instead of attachment | `file-delivery` config set to link | Run `openclaw config set messaging.file-delivery inline` |
| Bot sends "thinking..." messages | `message-style` not set to single-complete | Run `openclaw config set slack.message-style single-complete` |
| Cron successes flooding the channel | `cron-updates` not set to failures-only | Run `openclaw config set messaging.cron-updates failures-only` |
| Topic not found error | Topic was deleted, renamed, or ID changed | Re-create the topic and update the thread ID in TOOLS.md |
| WhatsApp messages not sending | WhatsApp plugin not enabled or chat ID wrong | Check `openclaw plugin list`, verify chat ID in TOOLS.md |
| Client has more groups than the 14 defaults | Client has custom topics for their workflow | Record them in TOOLS.md so other systems avoid posting there |

## Dependencies

- **Depends on:** `deploy-identity.md` (personality and tone affect how messages are written, deploy identity first so the agent's voice is set before it starts sending messages)
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
