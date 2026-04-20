# Deploy Earnings Reports

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-19 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" status was stale. `openclaw cron`, `openclaw config set`, `openclaw config` (for read via `openclaw config get`) all exist on 2026.4.15 per `audit/_baseline.md` §3 / §4C.7
- **`openclaw status` does NOT exist** — use `openclaw health` (top-level health probe) or `openclaw gateway status` (gateway-specific)
- **Scheduling**: `openclaw cron add` handles both recurring (`--cron "0 5 * * MON"`) and one-shot (`--at "2026-04-28 08:00"`) jobs — earnings release timing works natively
- **Dynamic one-time crons**: `openclaw cron add --delete-after-run` auto-cleans post-earnings
- **Model**: agent default (`agents.defaults.model` in `openclaw.json`). Override this skill's LLM routing via env var **`EARNINGS_NARRATIVE_MODEL`** (see `_authoring/_deploy-common.md` "Per-skill env-var overrides" for the pattern + examples). Applies to earnings narrative generation.
- **ClawHub alternatives**: `earnings-reader`, `earnings-calendar`, `stock-earnings-review` — see `audit/_baseline.md` §B if you'd rather install a community skill

## Purpose

Install an earnings report tracking system that gives a client automated, AI-generated narrative summaries of earnings releases for the stocks they follow. A weekly preview surfaces the upcoming earnings calendar, the client selects which companies to cover, and dynamic one-time cron jobs fire shortly after each release to post a concise narrative summary.

**When to use:** When the client follows stocks and wants automated earnings analysis delivered to their messaging channel.

**What this skill does:**
1. Configures an earnings watchlist of ticker symbols
2. Sets up a recurring weekly preview cron job (Sunday morning)
3. Creates the earnings preview and per-company narrative scripts
4. Enables dynamic one-time cron job creation timed to each company's earnings release
5. Delivers all output to the client's Earnings messaging topic

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Every variable used in this skill appears here.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${EARNINGS_TOPIC_ID}` | Client profile: messaging config (created during `deploy-messaging-setup.md`) | `694` |
| `${NOTIFICATION_CHANNEL}` | Client profile: preferred notification service | `telegram` |
| `${WATCHLIST}` | Client input: comma-separated ticker symbols `[HUMAN_INPUT]` | `AAPL,GOOG,MSFT,AMZN,NVDA,TSLA,META` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if any earnings cron jobs already exist:

**Remote:**
```
openclaw cron list 2>/dev/null | grep -i earnings
```

Expected: No output if this is a fresh install. If earnings jobs already exist, evaluate whether to update or skip.

Check if the earnings scripts directory exists:

**Remote:**
```
ls ${WORKSPACE}/scripts/earnings-preview.js ${WORKSPACE}/scripts/earnings-create-jobs.js 2>/dev/null || echo 'NOT_FOUND'
```

Expected: `NOT_FOUND` for a fresh install. If scripts exist, compare content before overwriting.

Confirm with the client:
- Does the client follow stocks? Which tickers? `[HUMAN_INPUT]`
- Does the client want weekly previews or on-demand only?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Watchlist | User-configured tickers `[HUMAN_INPUT]` | Client's investment focus changes; adjust list per their holdings or interests |
| Preview schedule | Sunday 9am in `${TIMEZONE}` | Client prefers a different day/time |
| Report style | Narrative (verdict, reaction, 2-3 takeaways) | Client prefers data tables, bullet points, or longer analysis |
| Delivery channel | Telegram topic `${EARNINGS_TOPIC_ID}` | Client uses Discord, Slack, or email instead |
| Earnings data source | Web research + AI analysis | Client has API access (Alpha Vantage, Yahoo Finance) for structured data |
| Dynamic cron jobs | One-time jobs with `deleteAfterRun: true` | Client prefers persistent quarterly-recurring jobs |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed (Earnings topic created, topic ID known)
- AI API access configured for generating narrative summaries

## What Gets Installed

| Component | Location | Description |
|-----------|----------|-------------|
| Weekly preview script | `${WORKSPACE}/scripts/earnings-preview.js` | Checks earnings calendar, filters to watchlist, posts preview |
| Dynamic job creator script | `${WORKSPACE}/scripts/earnings-create-jobs.js` | Creates one-time cron jobs per selected ticker |
| Weekly preview cron | `openclaw cron` — Sunday 9am | Recurring job that runs the preview script |
| Per-company one-time crons | `openclaw cron` — timed per release | Created dynamically, auto-delete after running |
| Earnings config | `openclaw config` — `earnings.*` keys | Watchlist, topic ID, report style preferences |

### How the System Works

1. **Sunday morning (recurring cron):** The weekly preview job runs, checks the earnings calendar for the upcoming week, filters to the client's watchlist, and posts a preview to the Earnings topic listing each company, their expected earnings date/time, and consensus estimates.

2. **Client selects tickers `[HUMAN_INPUT]`:** The client responds with which companies they want covered (all of them or a subset).

3. **Dynamic jobs created:** For each selected ticker, a one-time cron job is created timed to run shortly after the company's earnings release (typically 15-30 minutes after market close or before market open).

4. **Earnings narrative posted:** When each one-time job fires, it pulls earnings data and market reaction, generates a narrative summary, and posts it to the Earnings topic.

5. **Job auto-deletes:** Each one-time job removes itself from the cron system after running (`deleteAfterRun: true`).

### Narrative Summary Format

Each earnings report follows this structure:
- **Overall verdict:** Beat or miss on revenue and EPS
- **Market reaction:** How the stock moved in after-hours/pre-market
- **2-3 most interesting takeaways:** Guidance changes, segment surprises, management commentary
- **Style:** Written as prose, not data tables. Readable in 30 seconds.

## Steps

### Phase 1: Configure Earnings System

#### 1.1 Set the Earnings Watchlist `[HUMAN_INPUT]`

Ask the client which ticker symbols they want to track. This defines the universe of stocks the weekly preview will cover.

**Remote:**
```
openclaw config set earnings.watchlist '${WATCHLIST}'
```

Expected: Config updated confirmation. Verify with `openclaw config show earnings`.

If this fails: Check that `openclaw config` commands work at all. If not, verify OpenClaw is running with `openclaw health`.

If already exists: Compare the existing watchlist with the client's desired list. Update if different, skip if identical.

#### 1.2 Configure the Earnings Topic `[AUTO]`

Point the earnings system at the correct messaging topic for delivery.

**Remote:**
```
openclaw config set earnings.telegram-topic ${EARNINGS_TOPIC_ID}
```

Expected: Config updated. The earnings scripts will read this value to know where to post.

If this fails: Verify the topic ID is correct by checking the client profile or `deploy-messaging-setup.md` output.

If already exists: Compare the stored topic ID with `${EARNINGS_TOPIC_ID}`. Update if different, skip if identical.

> **Note:** If the client uses a non-Telegram channel, adapt this config key to match their notification system (e.g., Slack channel ID, Discord webhook URL).

### Phase 2: Create Scripts

#### 2.1 Write the Earnings Preview Script `[GUIDED]`

Create the script that checks the earnings calendar for the upcoming week and posts a preview filtered to the client's watchlist.

This script should:
- Read the watchlist from `openclaw config`
- Query the earnings calendar (upcoming week) for those tickers
- Format a preview message with: company name, ticker, expected date/time, consensus EPS and revenue estimates
- Post the preview to the Earnings topic (`${EARNINGS_TOPIC_ID}`)
- Support a `--manual` flag for on-demand execution

**Remote:**
```
cat > ${WORKSPACE}/scripts/earnings-preview.js << 'SCRIPT_EOF'
// Earnings Preview Script
// Reads watchlist from openclaw config, checks earnings calendar,
// posts preview to Earnings topic.
//
// Usage:
//   node earnings-preview.js           (cron mode)
//   node earnings-preview.js --manual  (on-demand test)
//
// The operator should implement the earnings calendar lookup
// and Telegram posting logic appropriate to the client's setup.
SCRIPT_EOF
```

> **Important:** The script content above is a placeholder skeleton. The operator must implement the actual earnings calendar lookup (web scraping, financial API, or AI research) and messaging integration. The implementation depends on the client's data sources and messaging platform.

Expected: File created at `${WORKSPACE}/scripts/earnings-preview.js`. Verify with `ls -la ${WORKSPACE}/scripts/earnings-preview.js`.

If this fails: Check that `${WORKSPACE}/scripts/` directory exists. Create it first if missing: `mkdir -p ${WORKSPACE}/scripts`.

If already exists: Compare content. If unchanged, skip. If different, back up as `earnings-preview.js.bak` and write the new version.

#### 2.2 Write the Dynamic Job Creator Script `[GUIDED]`

Create the script that takes selected tickers and creates one-time cron jobs timed to each company's earnings release.

This script should:
- Accept `--tickers 'AAPL,GOOG'` argument with the selected subset
- Look up each ticker's expected earnings release date and time
- Create a one-time cron job for each ticker via `openclaw cron add` with `deleteAfterRun: true`
- Each job runs a narrative summary script that pulls earnings data, generates a narrative, and posts to the Earnings topic

**Remote:**
```
cat > ${WORKSPACE}/scripts/earnings-create-jobs.js << 'SCRIPT_EOF'
// Dynamic Earnings Job Creator
// Takes selected tickers, looks up their earnings release times,
// creates one-time cron jobs timed to fire shortly after each release.
//
// Usage:
//   node earnings-create-jobs.js --tickers 'AAPL,GOOG'
//
// Each created job:
//   - Runs 15-30 min after the company's scheduled earnings release
//   - Generates a narrative summary (verdict, reaction, takeaways)
//   - Posts to the Earnings topic
//   - Has deleteAfterRun: true (auto-removes after execution)
//
// The operator should implement the earnings time lookup and
// cron job creation logic appropriate to the client's setup.
SCRIPT_EOF
```

Expected: File created at `${WORKSPACE}/scripts/earnings-create-jobs.js`.

If this fails: Same as 2.1 — check the scripts directory exists.

If already exists: Compare content. If unchanged, skip. If different, back up and write the new version.

### Phase 3: Schedule Weekly Preview

#### 3.1 Add the Weekly Preview Cron Job `[AUTO]`

Register the recurring cron job that runs the earnings preview every Sunday morning.

**Remote:**
```
openclaw cron add --name 'Weekly Earnings Preview' --schedule '0 9 * * 0' --tz '${TIMEZONE}' \
  --env EARNINGS_NARRATIVE_MODEL=openai-codex/gpt-5.4-nano \
  --command 'node ${WORKSPACE}/scripts/earnings-preview.js'
```

`EARNINGS_NARRATIVE_MODEL` routes the earnings narrative LLM calls. Omit to defer to the agent default. In `scripts/earnings-preview.js` (and in each one-time narrative job created by `earnings-create-jobs.js`):

```javascript
const MODEL = process.env.EARNINGS_NARRATIVE_MODEL || '';  // empty → agent default
const modelFlag = MODEL ? ['--model', MODEL] : [];
spawn('openclaw', ['agent', '--agent', 'main', ...modelFlag, '-m', prompt]);
```

When `earnings-create-jobs.js` creates the per-ticker one-time jobs via `openclaw cron add --delete-after-run`, it should also pass `--env EARNINGS_NARRATIVE_MODEL=<value>` through.

Expected: Cron job created. Verify with `openclaw cron list | grep -i 'earnings preview'`.

If this fails: Check that `openclaw cron add` works. Verify the cron system is active with `openclaw cron list`.

If already exists: Check for a duplicate with `openclaw cron list | grep -i 'earnings preview'`. If the schedule matches, skip. If different, remove the old one and re-add with the correct schedule.

> **Idempotency:** Always check for an existing "Weekly Earnings Preview" job before adding. Adding a duplicate would cause the preview to fire twice.

### Phase 4: Test the System

#### 4.1 Run a Manual Preview `[GUIDED]`

Test the preview script to verify it works end-to-end.

**Remote:**
```
node ${WORKSPACE}/scripts/earnings-preview.js --manual
```

Expected: A preview message listing upcoming earnings dates for watchlist stocks, posted to the Earnings topic (`${EARNINGS_TOPIC_ID}`).

If this fails: Check script implementation, check network access for earnings calendar data, check messaging credentials.

#### 4.2 Test Ticker Selection and Dynamic Job Creation `[HUMAN_INPUT]`

After the preview is posted, the client selects which tickers to cover. Create dynamic jobs for the selected tickers.

> **Ask the client:** "The earnings preview has been posted. Which companies would you like detailed coverage for? You can pick all or a subset."

**Remote:**
```
node ${WORKSPACE}/scripts/earnings-create-jobs.js --tickers '${SELECTED_TICKERS}'
```

> **Note:** `${SELECTED_TICKERS}` comes from the client's response — this is always a `[HUMAN_INPUT]` step. The operator substitutes the client's chosen tickers (e.g., `AAPL,GOOG`).

Expected: One-time cron jobs created in `~/.openclaw/cron/jobs.json` with `deleteAfterRun: true`, each timed to the corresponding company's earnings release.

If this fails: Check that the earnings-create-jobs script can look up earnings times. The earnings date/time for some companies may need manual specification if the calendar source is unavailable.

#### 4.3 Verify Dynamic Jobs Were Created `[AUTO]`

Confirm the one-time jobs are registered in the cron system.

**Remote:**
```
openclaw cron list | grep -i earnings
```

Expected: The recurring "Weekly Earnings Preview" job plus one-time per-company jobs for each selected ticker.

If this fails: Check `~/.openclaw/cron/jobs.json` directly for the job entries.

## Verification

Post-deployment checks to confirm everything is operational:

**Verify the weekly preview cron exists:**

**Remote:**
```
openclaw cron list | grep "Weekly Earnings Preview"
```

Expected: One entry showing Sunday 9am schedule in the client's timezone.

**Verify the earnings config:**

**Remote:**
```
openclaw config show earnings
```

Expected: Shows `watchlist` with the client's tickers and `telegram-topic` with `${EARNINGS_TOPIC_ID}`.

**Run a manual preview:**

**Remote:**
```
node ${WORKSPACE}/scripts/earnings-preview.js --manual
```

Expected: Preview message posted to the Earnings topic.

**After an actual earnings release (deferred verification):**
- Check the Earnings topic for the narrative summary
- Verify the one-time job has been removed from `~/.openclaw/cron/jobs.json`

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Configure watchlist | Prompt for tickers `[HUMAN_INPUT]` | Prompt for tickers `[HUMAN_INPUT]` | Prompt for tickers `[HUMAN_INPUT]` |
| 1.2 Configure topic | Execute silently | Execute silently | Show command, confirm |
| 2.1 Write preview script | Execute silently | Show script content, confirm | Show script, confirm each file |
| 2.2 Write job creator script | Execute silently | Show script content, confirm | Show script, confirm each file |
| 3.1 Add weekly cron | Execute silently | Confirm schedule details | Show command, confirm |
| 4.1 Run manual preview | Execute and report result | Execute and report result | Show command, confirm, report |
| 4.2 Select tickers | Prompt for selection `[HUMAN_INPUT]` | Prompt for selection `[HUMAN_INPUT]` | Prompt for selection `[HUMAN_INPUT]` |
| 4.3 Verify dynamic jobs | Execute silently | Execute and show results | Show command, confirm |

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No preview on Sunday | Cron job not scheduled or cron system inactive | Check `openclaw cron list` for the Weekly Earnings Preview. Verify cron system is active with `openclaw health`. |
| Missing earnings dates | Earnings calendar data source unavailable | The earnings calendar may need manual updating for some stocks. Check the data source API or use an alternative source. |
| One-time job didn't fire | Earnings date/time shifted after job was created | Companies sometimes reschedule earnings. Re-create the job with the updated time. |
| One-time job didn't auto-delete | `deleteAfterRun` flag missing from job | Check the job in `~/.openclaw/cron/jobs.json`. It should have `"deleteAfterRun": true`. Manually delete stale jobs. |
| Summary quality too low | Narrative prompt needs tuning | Adjust the prompt to emphasize the format: verdict, market reaction, 2-3 takeaways. More specific prompts produce better output. |
| Wrong messaging topic | Topic ID misconfigured | Verify with `openclaw config show earnings`. Update with `openclaw config set earnings.telegram-topic ${EARNINGS_TOPIC_ID}`. |
| Watchlist changes not reflected | Config not reloaded by next run | After updating the watchlist, the next weekly preview will use the new list. No restart needed. |
| Preview runs at wrong time | Timezone mismatch | Check the cron job timezone with `openclaw cron list`. Should match `${TIMEZONE}`. |

## Dependencies

- **Depends on:** `openclaw/install.md` (OpenClaw running), `deploy-messaging-setup.md` (Earnings topic created)
- **Independent of:** All other data systems
- **Pairs well with:** `deploy-advisory-council.md` (MarketAnalyst persona can reference earnings data for context)
