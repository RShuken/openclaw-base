# Deploy Earnings Reports

## Purpose

Install an earnings report tracking system. Weekly preview of upcoming earnings, then dynamically creates one-time cron jobs timed to each company's earnings release for narrative summaries.

**When to use:** When the user follows stocks and wants automated earnings analysis.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "openclaw cron list 2>/dev/null | grep -i earnings"
```
- Does the client follow stocks? Which tickers?
- Does the client want weekly previews or on-demand only?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Watchlist | User-configured tickers | Pre-set list, sector-based, or index-based | Client's investment focus |
| Preview schedule | Sunday 9am PST | Different day/time, client timezone | Client's schedule |
| Report style | Narrative (verdict, reaction, 2-3 takeaways) | Data tables, bullet points, longer analysis | Client's reading preference |
| Delivery channel | Telegram Earnings topic | Discord, Slack, email | Client's messaging platform |
| Earnings data source | Web research + AI analysis | Financial API (Alpha Vantage, Yahoo Finance) | Client has API access for structured data |
| Dynamic cron jobs | One-time jobs auto-delete after running | Persistent jobs that recur each quarter | Client prefers different scheduling model |

## Prerequisites

- `deploy-messaging-setup.md` completed (Telegram Earnings topic)
- AI API access for generating narrative summaries

## What Gets Installed

### Earnings System

| Component | Description |
|-----------|-------------|
| Weekly preview | Sunday 9am PST -- previews upcoming earnings for watchlist stocks |
| Ticker selection | User picks which companies to cover from the preview |
| Dynamic one-time crons | Created per company, timed to run right after their earnings release |
| Auto-cleanup | One-time jobs delete themselves after running (`deleteAfterRun: true`) |

### How It Works

1. **Sunday 9am PST:** The weekly preview job runs. It checks the earnings calendar for the upcoming week and filters to the user's watchlist stocks. Posts a preview to the Earnings Telegram topic listing each company, their expected earnings date/time, and consensus estimates.

2. **User selects tickers:** The user responds with which companies they want covered. This can be all of them or a subset.

3. **Dynamic jobs created:** For each selected ticker, a one-time cron job is created timed to run shortly after the company's earnings release (typically 15-30 minutes after market close or before market open, depending on the company).

4. **Earnings narrative posted:** When the one-time job fires, it pulls the earnings data, market reaction, and generates a narrative summary. Posted to the Earnings Telegram topic.

5. **Job auto-deletes:** The one-time job removes itself from the cron system after running.

### Narrative Summary Format

Each earnings report follows this structure:
- **Overall verdict:** Beat or miss on revenue and EPS
- **Market reaction:** How the stock moved in after-hours/pre-market
- **2-3 most interesting takeaways:** The notable things from the report (guidance changes, segment surprises, management commentary)
- **Narrative style:** Written as prose, not tables of numbers. Readable in 30 seconds.

### Telegram Delivery

- Earnings topic (ID: 694)
- Weekly preview posts Sunday mornings
- Individual earnings summaries post as each company reports

### Cron Schedule

| Job | Schedule | Type |
|-----|----------|------|
| Weekly Earnings Preview | Sunday 9am PST | Recurring |
| Per-company earnings summary | Timed to each company's release | One-time (auto-deletes) |

## Steps

### 1. Configure Earnings Watchlist

Define the list of ticker symbols the user follows.

```
cmd --session <id> --command "openclaw config set earnings.watchlist 'AAPL,GOOG,MSFT,AMZN,NVDA,TSLA,META'"
```

Adjust the ticker list to match the user's actual holdings or interests.

### 2. Set Up Weekly Preview Cron

```
cmd --session <id> --command "openclaw cron add --name 'Weekly Earnings Preview' --schedule '0 9 * * 0' --tz 'America/Los_Angeles' --command 'node ~/clawd/scripts/earnings-preview.js'"
```

This runs every Sunday at 9am PST.

### 3. Configure Telegram Earnings Topic

```
cmd --session <id> --command "openclaw config set earnings.telegram-topic 694"
```

### 4. Run a Manual Preview

Test the preview for the upcoming week:

```
cmd --session <id> --command "node ~/clawd/scripts/earnings-preview.js --manual"
```

Expected: A preview message listing upcoming earnings dates for watchlist stocks, posted to the Earnings topic.

### 5. Select Tickers and Create Dynamic Jobs

After the preview is posted, select tickers to cover. This creates the one-time cron jobs:

```
cmd --session <id> --command "node ~/clawd/scripts/earnings-create-jobs.js --tickers 'AAPL,GOOG'"
```

Expected: One-time cron jobs created in `~/.openclaw/cron/jobs.json` timed to each company's earnings release.

### 6. Verify Dynamic Jobs

```
cmd --session <id> --command "openclaw cron list | grep -i earnings"
```

Expected: The weekly preview job plus any one-time per-company jobs.

## Verification

Check the system is operational:

```
# Verify weekly preview cron exists
openclaw cron list | grep "Weekly Earnings Preview"

# Run a manual preview
node ~/clawd/scripts/earnings-preview.js --manual

# Check for dynamic jobs after ticker selection
cat ~/.openclaw/cron/jobs.json | grep -i earnings
```

Expected:
- Weekly preview cron shows in the cron list with Sunday 9am PST schedule
- Manual preview produces a readable earnings calendar for watchlist stocks
- After selecting tickers, one-time jobs appear in `jobs.json` with `deleteAfterRun: true`

After an actual earnings release:
- Check the Earnings Telegram topic for the narrative summary
- Verify the one-time job has been removed from `jobs.json`

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No preview on Sunday | Cron job not scheduled or not running | Check `openclaw cron list` for the Weekly Earnings Preview. Verify cron system is active. |
| Missing earnings dates | Earnings calendar data source unavailable or outdated | The earnings calendar may need manual updating for some stocks. Check the data source API. |
| One-time job didn't fire | Job timed wrong (earnings date/time shifted) | Companies sometimes change their earnings date. Re-create the job with the updated time. |
| One-time job didn't auto-delete | `deleteAfterRun` flag missing | Check the job in `jobs.json`. It should have `"deleteAfterRun": true`. Manually delete stale jobs. |
| Summary quality too low | Prompt needs tuning | Adjust the narrative prompt to emphasize the format: verdict, market reaction, 2-3 takeaways. More specific prompts produce better output. |
| Wrong Telegram topic | Topic ID misconfigured | Verify earnings topic is 694 in `openclaw config show earnings`. |
| Watchlist changes not reflected | Config not reloaded | After updating the watchlist, the next weekly preview will use the new list. No restart needed. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md` (Telegram Earnings topic, ID: 694)
- **Independent of:** All other data systems
- **Pairs well with:** `deploy-advisory-council.md` (MarketAnalyst persona can reference earnings data for context)
