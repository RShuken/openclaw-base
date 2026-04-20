# Deploy Telegram Forum

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — uses raw Telegram Bot API throughout, version-proof regardless of OpenClaw CLI changes
- **Why raw API**: Forum/Topics mode configuration (creating topics, setting permissions) isn't exposed through `openclaw channels` as of 2026.4.15. Raw Bot API is the right tool for the setup phase. Post-setup message delivery can use either `openclaw channels send` or raw Bot API.
- **Model**: agent default (Codex-first); do not hardcode
- **ClawHub alternative**: `agent-telegram` covers basic Telegram skill install, but forum-mode setup is generally a one-time operator task this skill documents clearly

## Purpose

Enable Telegram Forum/Topics mode for a bot's group, verify bot permissions, create starter topic threads, and configure the group for organized conversations. This turns a flat Telegram group into a threaded workspace where the client can start topic-specific conversations with the bot.

**When to use:** Setting up a new client's Telegram group for the first time, migrating a basic group to forum mode, or adding topic threads to an existing forum group.

**What this skill does:**
1. Guides the client through enabling Topics/Forum mode in Telegram UI
2. Discovers the group chat ID via the Telegram Bot API
3. Verifies the bot has `can_manage_topics` admin permission
4. Creates named topic threads with color coding via `createForumTopic`
5. Tests message delivery to each thread via `sendMessage`
6. Adds the group to the bot's allowlist
7. Records topic thread IDs in the client profile

**Key difference from other skills:** All Telegram operations use direct Bot API calls (`curl` to `api.telegram.org/bot<TOKEN>/...`), not CLI abstractions.

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally. Bot API calls (`curl` to `api.telegram.org`) run from the operator's terminal — the bot token is read from the client's config but the API calls themselves run locally to avoid depending on curl/network on the client machine.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${BOT_TOKEN}` | Client config: `channels.telegram.botToken` in openclaw.json | `7123456789:AAF...` |
| `${GROUP_ID}` | Discovered in Phase 2 via `getUpdates` or client profile | `-1001234567890` |
| `${ALLOWFROM_PATH}` | Client config dir: `~/.openclaw/credentials/telegram-default-allowFrom.json` | (platform-dependent) |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Read the bot token from the client's openclaw config:

**Remote (Windows):**
```
(Get-Content $env:USERPROFILE\.openclaw\openclaw.json -Raw | ConvertFrom-Json).channels.telegram.botToken
```

**Remote (macOS/Linux):**
```
cat ~/.openclaw/openclaw.json | python3 -c "import sys,json; print(json.load(sys.stdin)['channels']['telegram']['botToken'])"
```

Expected: A bot token string like `7123456789:AAF...`. Record this as `${BOT_TOKEN}`.

Check the bot identity:

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getMe"
```

Expected: JSON with the bot's username, name, and capabilities. Confirm `can_join_groups: true`.

**Discovery questions to answer before proceeding:**
- Does the client have a Telegram group already, or do they need to create one?
- Is the bot already added to the group as admin?
- Is Forum/Topics mode already enabled, or does the client need to enable it?
- What starter topics does the client want? (Use recommended defaults or customize)

## Adaptation Points

| Component | Recommended Default | Customize When... |
|-----------|-------------------|-------------------|
| Forum/Topics mode | Enabled (supergroup with topics) | Client explicitly prefers a flat group (skip this skill) |
| Starter topics | General, Quick Questions, Research, Tasks & Reminders, Ideas & Brainstorm | Client has specific topic names for their workflow |
| Topic colors | Blue, Violet, Green, Yellow | Client prefers different colors or no color coding |
| Welcome message | Sent to General with topic descriptions | Client prefers no welcome message or a custom one |
| Allowlist update | Add group chat ID to allowFrom | Client manages allowlist differently |
| Gateway restart | Stop and restart after allowlist change | Client has a different restart method or the gateway auto-reloads |

## Prerequisites

- OpenClaw agent installed and running with Telegram plugin enabled
- Bot token configured in `openclaw.json` under `channels.telegram.botToken`
- A Telegram group exists (or client will create one) with the bot added
- Active remote session to the client's machine (for config reads and gateway management)

## What Gets Installed

This skill does not install files or databases. It configures:

- **Telegram group:** Forum/Topics mode enabled (manual step by client)
- **Bot permissions:** `can_manage_topics` verified
- **Topic threads:** Created via Bot API with names and color codes
- **Allowlist:** Group chat ID added to `telegram-default-allowFrom.json`
- **Client profile:** Updated with group ID, topic thread IDs, and forum status

### Telegram Bot API Methods Used

| Method | Purpose | Docs |
|--------|---------|------|
| `getMe` | Verify bot identity | https://core.telegram.org/bots/api#getme |
| `getUpdates` | Discover group chat ID from recent messages | https://core.telegram.org/bots/api#getupdates |
| `getChat` | Verify group is a supergroup with forum mode | https://core.telegram.org/bots/api#getchat |
| `getChatMember` | Check bot's admin permissions | https://core.telegram.org/bots/api#getchatmember |
| `createForumTopic` | Create a named topic thread | https://core.telegram.org/bots/api#createforumtopic |
| `deleteForumTopic` | Remove a topic thread (cleanup) | https://core.telegram.org/bots/api#deleteforumtopic |
| `sendMessage` | Send a message to a specific thread | https://core.telegram.org/bots/api#sendmessage |

### Topic Color Codes

Telegram supports a fixed set of icon colors for forum topics:

| Color | Code |
|-------|------|
| Blue | `7322096` |
| Yellow | `16766590` |
| Violet | `13338331` |
| Green | `9367192` |
| Red | `16749490` |
| Orange | `16478047` |

## Steps

### Phase 1: Discovery

#### 1.1 Read Bot Token from Client Config `[AUTO]`

Read the bot token from the client's openclaw configuration.

**Remote (Windows):**
```
(Get-Content $env:USERPROFILE\.openclaw\openclaw.json -Raw | ConvertFrom-Json).channels.telegram.botToken
```

**Remote (macOS/Linux):**
```
python3 -c "import json; print(json.load(open('$HOME/.openclaw/openclaw.json'))['channels']['telegram']['botToken'])"
```

Expected: Bot token string. Record as `${BOT_TOKEN}`.

If this fails: Telegram plugin is not configured. The bot token must be set in openclaw.json first. Check if the Telegram plugin is enabled: look for `channels.telegram.enabled: true` in the config.

#### 1.2 Verify Bot Identity `[AUTO]`

Confirm the bot is reachable and has the right capabilities.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getMe"
```

Expected: JSON response with `ok: true`. Record the bot's `id`, `username`, and `first_name`. Confirm `can_join_groups: true`.

If this fails: Bot token is invalid or revoked. The client needs to get a fresh token from @BotFather.

#### 1.3 Check Existing Allowlist `[AUTO]`

Read the current allowlist to see if a group is already configured.

**Remote (Windows):**
```
Get-Content $env:USERPROFILE\.openclaw\credentials\telegram-default-allowFrom.json -Raw
```

**Remote (macOS/Linux):**
```
cat ~/.openclaw/credentials/telegram-default-allowFrom.json
```

Expected: JSON with an `allowFrom` array. May contain user IDs (DM pairing) but no group IDs yet.

If this fails: The file may not exist if Telegram was never paired. Create it during Phase 5.

### Phase 2: Group Discovery

#### 2.1 Attempt to Find Group from Updates `[GUIDED]`

The Bot API `getUpdates` returns recent messages the bot received. The group chat ID is in the `chat.id` field. However, if the openclaw gateway is running, it consumes updates via long-polling and `getUpdates` will return empty.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?offset=-1&limit=5"
```

Expected: If gateway is NOT running, returns recent updates with group chat info. Look for entries where `chat.type` is `supergroup` and `chat.is_forum` is `true`.

If empty (gateway is consuming updates): Proceed to Step 2.2.

If the client already knows their group ID (e.g., from a previous session or profile): Skip to Phase 3.

#### 2.2 Stop Gateway to Capture Updates `[GUIDED]`

If the gateway is consuming all updates, temporarily stop it so we can capture Jeff's next message.

First, identify the gateway processes:

**Remote (Windows):**
```
Get-Process -Name node -ErrorAction SilentlyContinue | Select-Object Id, ProcessName, @{N="CmdLine";E={(Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine}} | Format-Table -AutoSize -Wrap
```

**Remote (macOS/Linux):**
```
pgrep -af 'openclaw.*gateway'
```

Expected: One or more node processes running the openclaw gateway.

Stop the gateway:

**Remote (Windows):**
```
Stop-Process -Id <PID1>,<PID2> -Force -ErrorAction SilentlyContinue
```

**Remote (macOS/Linux):**
```
pkill -f 'openclaw.*gateway'
```

Expected: Gateway processes terminated.

> **Important:** Tell the client to send a message in the target group NOW (anything, even "hello"). The gateway is stopped so the Bot API will queue the update for us.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?limit=10&timeout=5"
```

Expected: Updates appear with the client's message. Find the entry where `message.chat.type` is `supergroup`. Record `chat.id` as `${GROUP_ID}` and note `chat.is_forum` (true if forum mode is already enabled).

If this fails: The client may not have sent a message yet. Wait and retry. If no updates appear after 30 seconds, the client should check that the bot is still a member of the group.

### Phase 3: Verify Group and Bot Permissions

#### 3.1 Verify Forum Mode `[AUTO]`

Check that the group is a supergroup with forum mode enabled.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getChat?chat_id=${GROUP_ID}"
```

Expected: Response shows `type: "supergroup"` and `is_forum: true`.

If `is_forum` is `false` or missing: The client needs to enable Topics mode manually. Provide the client walkthrough (see "Client Walkthrough" section below).

#### 3.2 Verify Bot Admin Permissions `[AUTO]`

Check that the bot has the required admin permissions, especially `can_manage_topics`.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getChatMember?chat_id=${GROUP_ID}&user_id=<BOT_ID>"
```

> **Important:** Replace `<BOT_ID>` with the bot's numeric ID from Step 1.2.

Expected: Response shows `status: "administrator"` with `can_manage_topics: true`.

Required permissions:
- `can_manage_topics: true` — create, close, rename threads
- `can_manage_chat: true` — general admin functions
- `can_delete_messages: true` — cleanup

If `can_manage_topics` is `false`: The client needs to update the bot's admin permissions. Provide the permissions walkthrough.

If `status` is not `administrator`: The client needs to promote the bot to admin. Guide them through group settings > Administrators > Add the bot.

### Phase 4: Client Walkthroughs (If Needed)

These are text instructions to send to the client. Only use them if Phase 3 identified missing prerequisites.

#### 4.1 Enable Topics/Forum Mode `[HUMAN_INPUT]`

Send this to the client if `is_forum` is false:

> **Enable Topics in your Telegram group:**
> 1. Open the group in Telegram
> 2. Tap the group name at the top to open Group Info
> 3. Tap Edit (pencil icon, top right)
> 4. Scroll down and find "Topics" — toggle it ON
> 5. Tap the checkmark to save
>
> If you don't see "Topics": Your group may need to be converted to a Supergroup first. Telegram usually auto-converts when you enable Topics. If the toggle isn't visible, go to Group Type and switch to Supergroup first.

After the client confirms, re-run Step 3.1 to verify.

#### 4.2 Update Bot Admin Permissions `[HUMAN_INPUT]`

Send this to the client if `can_manage_topics` is false:

> **Update bot admin permissions:**
> 1. In Group Info, tap "Administrators"
> 2. Find the bot in the admin list and tap its name
> 3. Make sure these permissions are ON:
>    - Manage Topics (required to create/close/rename threads)
>    - Send Messages
>    - Delete Messages (optional but helpful)
> 4. "Manage Topics" only appears after Topics mode is enabled
> 5. Tap Save / Done

After the client confirms, re-run Step 3.2 to verify.

### Phase 5: Create Topics

#### 5.1 Plan Topics with Client `[GUIDED]`

Present the recommended starter topics and ask the client which ones they want:

| # | Topic | Color | Purpose |
|---|-------|-------|---------|
| 1 | Quick Questions | Blue (7322096) | Fast answers, one-off questions |
| 2 | Research | Violet (13338331) | Deep dives, analysis, longer conversations |
| 3 | Tasks & Reminders | Green (9367192) | Things to track or do |
| 4 | Ideas & Brainstorm | Yellow (16766590) | Creative thinking, planning |

The client can customize names, add topics, or skip any of these. The "General" topic is created automatically by Telegram when forum mode is enabled.

Expected: A confirmed list of topics to create with names and colors.

#### 5.2 Create Topic Threads `[AUTO]`

For each topic, create a forum topic via the Bot API.

**Operator (repeat for each topic):**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/createForumTopic" \
  -d "chat_id=${GROUP_ID}" \
  -d "name=<TOPIC_NAME>" \
  -d "icon_color=<COLOR_CODE>"
```

Expected: Each call returns `ok: true` with a `message_thread_id`. Record each thread ID.

If this fails: Check that `can_manage_topics` is true (Step 3.2). If the bot lost admin privileges, have the client re-promote it.

If already exists: There's no `listForumTopics` API method, so you can't check for duplicates by name. If you know a topic already exists (from a previous run), skip creating it. If in doubt, create it — duplicate names are allowed in Telegram.

#### 5.3 Send Welcome Message `[AUTO]`

Send a welcome message in the General topic that describes the available topics.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -d "chat_id=${GROUP_ID}" \
  --data-urlencode "text=<WELCOME_MESSAGE>" \
  -d "parse_mode=Markdown"
```

> **Note:** Sending to General means omitting the `message_thread_id` parameter (or using `message_thread_id=1` which is the General thread).

The welcome message should list each created topic and its purpose. Customize based on the bot's identity and the client's preferences.

Expected: Message appears in the General topic.

If this fails: Check `parse_mode` — Markdown can fail on special characters. Try without `parse_mode` or use `HTML` instead.

#### 5.4 Test Each Topic `[AUTO]`

Send a test message to each created topic to verify routing works.

**Operator (repeat for each topic):**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -d "chat_id=${GROUP_ID}" \
  -d "message_thread_id=<THREAD_ID>" \
  -d "text=Test message in <TOPIC_NAME>"
```

Expected: Messages appear in the correct topics. Ask the client to confirm they see them.

If this fails: The thread ID may be wrong. Re-check the `createForumTopic` response for the correct `message_thread_id`.

#### 5.5 Clean Up Test Messages (Optional) `[AUTO]`

If you created a test-only topic (like "Test Thread"), delete it.

**Operator:**
```
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/deleteForumTopic" \
  -d "chat_id=${GROUP_ID}" \
  -d "message_thread_id=<TEST_THREAD_ID>"
```

Expected: The test topic is removed from the group.

### Phase 6: Configuration

#### 6.1 Add Group to Allowlist `[GUIDED]`

Add the group chat ID to the bot's allowlist so the openclaw agent responds to messages there.

**Remote (Windows):**
```powershell
$f = "$env:USERPROFILE\.openclaw\credentials\telegram-default-allowFrom.json"
$j = Get-Content $f -Raw | ConvertFrom-Json
if ($j.allowFrom -notcontains "<GROUP_ID>") {
    $j.allowFrom += "<GROUP_ID>"
    $j | ConvertTo-Json -Depth 5 | Set-Content $f -Encoding UTF8
}
```

**Remote (macOS/Linux):**
```bash
python3 -c "
import json
f = '$HOME/.openclaw/credentials/telegram-default-allowFrom.json'
d = json.load(open(f))
gid = '<GROUP_ID>'
if gid not in d['allowFrom']:
    d['allowFrom'].append(gid)
    json.dump(d, open(f, 'w'), indent=2)
    print('Added')
else:
    print('Already present')
"
```

> **Important:** Replace `<GROUP_ID>` with the actual group chat ID (e.g., `-1003532553361`) before sending.

Expected: The allowFrom array now includes both the client's user ID and the group chat ID.

If this fails: Check that the credentials directory and file exist. If missing, create the file with the correct structure: `{"version": 1, "allowFrom": ["<USER_ID>", "<GROUP_ID>"]}`.

#### 6.2 Restart Gateway `[GUIDED]`

The gateway must be restarted to pick up the new allowlist. If it was stopped in Phase 2, start it now. If it's still running, restart it.

**Remote (Windows):**
```
Start-Process -FilePath "C:\Program Files\nodejs\node.exe" -ArgumentList "<OPENCLAW_MJS_PATH>","gateway" -WindowStyle Hidden
```

> **Important:** Replace `<OPENCLAW_MJS_PATH>` with the actual path (e.g., `C:\Users\jeffh\AppData\Roaming\npm\node_modules\openclaw\openclaw.mjs`). Find it by checking the process command line from Phase 2.2.

**Remote (macOS/Linux):**
```
nohup openclaw gateway > /dev/null 2>&1 &
```

Expected: Gateway process starts. Verify by checking running processes.

If this fails: Check that node and openclaw are on PATH. Use the full path to both if needed.

### Phase 7: Post-Deployment

#### 7.1 Update Client Profile `[AUTO]`

Update `clients/<name>.md` with:

- Telegram group name and chat ID
- Forum mode status
- Bot admin permissions confirmed
- List of created topics with thread IDs
- Any custom topics the client added
- "Telegram Forum deployed on [date]"

#### 7.2 Verify Everything Works `[AUTO]`

Ask the client to:
1. Open the group in Telegram and confirm all topics are visible
2. Send a message in one of the topics to confirm Joi responds
3. Try creating a new topic themselves (they can do this as group admin)

If Joi does not respond: Check that the gateway is running and the group is in the allowlist. Check gateway logs for errors.

## Verification

After deployment, the forum setup is confirmed working when:

1. **Group is a supergroup** with `is_forum: true`
2. **Bot has `can_manage_topics: true`** permission
3. **All requested topics exist** and received test messages
4. **Group is in the allowlist** in `telegram-default-allowFrom.json`
5. **Gateway is running** and the bot responds to messages in topics
6. **Client confirms** they can see topics and Joi responds

**Operator verification commands:**

```bash
# Verify forum mode
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getChat?chat_id=${GROUP_ID}" | python3 -c "import sys,json; d=json.load(sys.stdin)['result']; print(f'Forum: {d.get(\"is_forum\")}, Type: {d.get(\"type\")}')"

# Verify bot permissions
curl -sS "https://api.telegram.org/bot${BOT_TOKEN}/getChatMember?chat_id=${GROUP_ID}&user_id=<BOT_ID>" | python3 -c "import sys,json; m=json.load(sys.stdin)['result']; print(f'Status: {m[\"status\"]}, Manage Topics: {m.get(\"can_manage_topics\")}')"
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `getUpdates` returns empty | Gateway is consuming updates via long-polling | Stop the gateway temporarily (Phase 2.2), have the client send a message, then call `getUpdates` |
| `createForumTopic` returns 400 | Bot lacks `can_manage_topics` permission or group is not a forum | Verify permissions (Phase 3) and forum mode (Phase 3.1) |
| `sendMessage` with `message_thread_id` fails | Thread ID is wrong or topic was deleted | Re-check the `createForumTopic` response or ask the client to confirm the topic exists |
| Bot doesn't respond in group | Group not in allowlist or gateway not running | Check allowlist (Phase 6.1) and gateway status (Phase 6.2) |
| "Topics" toggle not visible in group settings | Group is a basic group, not a supergroup | Convert to supergroup first (Telegram usually does this automatically when Topics is enabled) |
| Bot can't be promoted to admin | Bot was added to the group by someone other than the client | The group creator or an existing admin must promote the bot |
| Gateway won't restart | Port conflict or node not found | Check if another gateway process is already running. Use full path to node if not on PATH |
| `openclaw config` blocks the PTY | The config command launches an interactive wizard | NEVER run `openclaw config` (without args) via remote session. Read the config file directly instead |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| Phase 1: Discovery (1.1-1.3) | Execute all silently | Execute all silently | Confirm each command |
| Phase 2: Group Discovery (2.1-2.2) | Execute, stop gateway if needed | Confirm before stopping gateway | Confirm each command |
| Phase 3: Verification (3.1-3.2) | Execute all silently | Execute all silently | Confirm each command |
| Phase 4: Client Walkthroughs | Always HUMAN_INPUT | Always HUMAN_INPUT | Always HUMAN_INPUT |
| Phase 5.1: Plan Topics | Execute defaults | Confirm topic list | Confirm each topic |
| Phase 5.2-5.5: Create and Test | Execute all | Confirm before creating | Confirm each topic |
| Phase 6: Configuration | Execute | Confirm before allowlist change | Confirm each command |
| Phase 7: Post-deployment | Execute | Execute | Show changes, confirm |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw must be installed with Telegram plugin enabled and bot token configured)
- **Required by:** `deploy-messaging-setup.md` (this skill sets up the forum infrastructure that messaging-setup's topic routing table builds on)
- **Related:** `deploy-identity.md` (bot personality affects welcome message tone), `deploy-messaging-setup.md` (creates the TOOLS.md routing table referencing the topics created here)
