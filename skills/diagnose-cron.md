# Diagnose & Fix Cron Jobs

**Purpose:** Troubleshoot and fix cron jobs that aren't firing on any client machine. Run this skill against any client to detect common cron failures.

**When to use:** Client reports morning briefing never arrives, nightly sprint didn't run, or any scheduled task silently fails.

---

## Known Issues Registry

| Issue | Affected Versions | Symptoms | Root Cause | Fix |
|-------|-------------------|----------|------------|-----|
| Cron silently advances | 2026.2.3 - 2026.2.5 | `nextRunAtMs` updates but no run record | Scheduler recomputes next time without executing | Upgrade to >= 2026.2.6-3 |
| Cron timer stops overnight | < 2026.3.x | Jobs stop firing during low-activity hours | `armTimer()` gets undefined `nextRunAtMs`, stops scheduler | Upgrade to >= 2026.3.x (has 60s maintenance fallback) |
| Telegram delivery silent fail | 2026.2.6 - 2026.2.22 | Run says `delivered: true` but no Telegram message | `delivery.channel` defaults to "last" which has no route in isolated sessions | Set explicit `--channel telegram --to "<chatId>:topic:<topicId>"` |
| Webhook drops on container restart | All (container deploys) | Webhook URL is blank, no messages received | Gateway didn't re-register webhook after container eviction/restart | Add explicit `setWebhook` curl in startup script |
| Long-polling dies on sleep | All (container/VPS) | Agent stops receiving messages after idle period | Poll connection drops when process sleeps or container evicts | Switch to webhook mode: set `webhookUrl` in config |
| Timezone mismatch | All | Jobs fire at wrong time | Host timezone differs from intended timezone | Always pass `--tz "America/Denver"` (or client's timezone) explicitly |
| Vague payload | All | Job fires but agent does nothing useful | Message too generic ("check on things") | Write specific, actionable instructions with concrete steps |

---

## Diagnostic Procedure

Run these steps IN ORDER on the client machine. Each step either passes or reveals the specific failure.

### Step 1: Check OpenClaw Version

```bash
openclaw --version
```

**Pass:** Version >= 2026.3.13
**Fail:** Any version < 2026.2.6-3 has known cron bugs

**Fix:**
```bash
npm update -g openclaw
openclaw gateway restart
```

### Step 2: Check Gateway is Running

```bash
# Check if gateway process is alive
pgrep -f "openclaw gateway" || echo "GATEWAY NOT RUNNING"

# If using launchd (macOS):
launchctl list | grep openclaw

# Check gateway logs for errors:
openclaw gateway logs 2>&1 | tail -20
```

**Pass:** Gateway process found and running
**Fail:** No process = no cron scheduler (cron runs INSIDE the gateway)

**Fix:**
```bash
openclaw gateway restart
# Or for launchd:
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

### Step 3: Check Telegram Webhook Status

```bash
# Get bot token from config
BOT_TOKEN=$(node -e "const c=JSON.parse(require('fs').readFileSync(require('os').homedir()+'/.openclaw/openclaw.json','utf8')); console.log(c.channels?.telegram?.botToken || 'NOT_SET')")
echo "Bot token: ${BOT_TOKEN:0:10}..."

# Check webhook
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getWebhookInfo" | python3 -m json.tool
```

**Pass:** `url` is non-empty and matches the expected endpoint, no recent errors
**Fail if `url` is empty:** Webhook not registered. Messages can't reach the agent.
**Fail if `last_error_message` present:** Webhook URL is unreachable.

**Fix (for container/VPS with reverse proxy):**
```bash
# Register webhook manually
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/setWebhook" \
  -d "url=https://YOUR_EXTERNAL_URL/telegram-webhook" \
  -d "secret_token=YOUR_SECRET"

# Add webhookUrl to config:
# channels.telegram.webhookUrl = "https://YOUR_EXTERNAL_URL/telegram-webhook"
# channels.telegram.webhookSecret = "YOUR_SECRET"
```

**Fix (for local/home machine — use long-polling):**
```bash
# Delete any stale webhook to enable long-polling
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/deleteWebhook"
# Gateway will auto-start long-polling on next restart
openclaw gateway restart
```

### Step 4: List Cron Jobs

```bash
openclaw cron list
```

**Pass:** Expected jobs listed with `enabled: true`
**Fail if empty:** No jobs registered
**Fail if jobs show but status is "disabled" or "error":** Re-enable or recreate

**Fix:**
```bash
# Example: re-add a morning briefing
openclaw cron add \
  --name "Morning Briefing" \
  --cron "0 7 * * *" \
  --tz "America/Denver" \
  --session isolated \
  --message "Run the daily briefing. Summarize overnight ticket activity, CRM updates, and post to Telegram." \
  --announce \
  --channel telegram \
  --to "-CHATID:topic:TOPICID"
```

### Step 5: Check Run History

```bash
# List recent runs for a specific job
openclaw cron runs --id <jobId> --limit 10

# Check all jobs' last run
openclaw cron status
```

**Pass:** Recent runs with `status: "ok"` or `delivered: true`
**Fail if no runs:** Job registered but never fired (version bug or gateway not running)
**Fail if runs exist but `delivered: false`:** Delivery target issue

### Step 6: Test Fire a Job

```bash
# Manually trigger a job right now
openclaw cron run <jobId>

# Watch for output in Telegram within 30 seconds
```

**Pass:** Message appears in Telegram
**Fail:** If no message appears, the delivery channel is broken. Check Step 3.

### Step 7: Check Cron Persistence

```bash
# Verify jobs.json exists and is valid
cat ~/.openclaw/cron/jobs.json | python3 -m json.tool | head -20

# Check for run history files
ls -la ~/.openclaw/cron/runs/ 2>/dev/null || echo "No run history directory"
```

**Pass:** Valid JSON with jobs listed
**Fail:** Missing or corrupt file = jobs lost on gateway restart

---

## Container-Specific Checks (Cloudflare/Docker)

### Check A: Container is alive and gateway started

```bash
# From the operator worker:
curl -H "Authorization: Bearer $TOKEN" \
  "https://WORKER_URL/debug/exec" \
  -d '{"cmd":"openclaw --version && openclaw cron list"}'
```

### Check B: Webhook registered from inside container

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://WORKER_URL/debug/exec" \
  -d '{"cmd":"curl -s https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getWebhookInfo"}'
```

### Check C: R2 config isn't overwriting patched config

The R2 sync runs every 30 seconds and syncs ~/.openclaw/ TO R2. But on restart, R2 restores to ~/.openclaw/. If the old config (without webhook settings) is in R2, it will overwrite the startup patching on next restart.

**Fix:** After config patching, the R2 sync will eventually push the new config to R2. But if there's a timing issue, manually sync:
```bash
rclone sync /root/.openclaw/ "r2:BUCKET/openclaw/" --transfers=16 --fast-list
```

---

## Prevention Checklist (For New Installs)

Before finishing any client install, verify:

- [ ] OpenClaw version >= 2026.3.13
- [ ] Gateway is running (process alive, launchd/systemd configured)
- [ ] Telegram webhook is registered (or long-polling is working)
- [ ] Cron jobs are listed in `openclaw cron list`
- [ ] Test fire at least one cron job and confirm Telegram delivery
- [ ] Timezone explicitly set on all cron jobs
- [ ] For containers: webhook registration in startup script
- [ ] For containers: keep-alive cron prevents container eviction
- [ ] Run history check: `openclaw cron runs --id <jobId>` shows at least 1 run

---

## Adaptation Points

| Setting | Default | Client-Specific |
|---------|---------|-----------------|
| Timezone | America/Denver | Client's local timezone |
| Telegram chat ID | `<CLIENT_CHAT_ID>` (e.g. `-100xxxxxxxxxx`) | Client's group chat ID |
| Telegram topic ID | `<CLIENT_TOPIC_ID>` (e.g. `156` for a Sprint topic) | Client's sprint/briefing topic ID |
| Sprint window | 22:00-06:00 | Client's preferred overnight hours |
| Briefing time | 07:00 | Client's preferred wake time |
| OpenClaw version | 2026.4.15+ | Always use latest stable (live-verified on 4C — see audit/_baseline.md §1) |
| Webhook vs polling | Webhook (containers) | Polling OK for local/home machines |
