# Deploy Social Media Tracker

## Purpose

Track daily performance snapshots across YouTube, Instagram, X/Twitter, and TikTok. Stores per-video/per-post analytics in SQLite. Generates charts. Feeds data into daily briefing and business advisory council.

**When to use:** When the user has social media accounts they want to track.

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**
```
cmd --session <id> --command "ls ${WORKSPACE}/tools/social-tracker/db/ 2>/dev/null"
cmd --session <id> --command "python3 --version 2>/dev/null"
cmd --session <id> --command "gog auth status 2>/dev/null"
```
- Which social platforms does the client actually use? Only deploy trackers for platforms they're on.
- Are the required API tokens available for each platform?
- Is Python 3 available?

## Adaptation Points

The defaults below work for most installs. Adapt when the client profile says otherwise.

| Component | Default | Alternative | When to Adapt |
|-----------|---------|-------------|---------------|
| Platforms tracked | YouTube + Instagram + X/Twitter + TikTok | Any subset | Client only uses some platforms. Deploy only what's needed. |
| YouTube auth | Google OAuth via `gog` | YouTube Data API key only (less data) | Client has limited Google scopes |
| Instagram auth | Meta Graph API long-lived token | Skip Instagram | Client doesn't use Instagram or can't get API access |
| X/Twitter auth | OAuth 1.0a (4 tokens) | App-only bearer token (less data), or skip | Client has limited X API access |
| TikTok tracking | Web scraping (no API) | Skip TikTok | Client doesn't use TikTok |
| Collection schedule | 1:00-1:45am PST staggered | Adjust timezone, different times | Client's timezone |
| Chart generation | Dark theme PNGs | Light theme, skip charts | Client preference |
| Database path | `${WORKSPACE}/tools/social-tracker/db/` | Custom path | Non-standard workspace |

## Prerequisites

- `deploy-messaging-setup.md` completed
- Python 3 available
- YouTube: Google OAuth token (`gog auth login` for YouTube Data API + Analytics API)
- Instagram: Meta Graph API long-lived token (INSTAGRAM_ACCESS_TOKEN or configured in .env)
- X/Twitter: OAuth 1.0a tokens (X_API_KEY, X_API_SECRET, X_ACCESS_TOKEN, X_ACCESS_TOKEN_SECRET)
- TikTok: No API needed (web scraping for follower count)

## What Gets Installed

### Databases

| Database | Location | Purpose |
|----------|----------|---------|
| YouTube Analytics | `tools/social-tracker/db/views.db` | Per-video views, watch time, avg view duration/percentage, likes, dislikes, comments, shares, subs gained/lost, thumbnail impressions/CTR + video metadata |
| Social Growth | `tools/social-tracker/db/social_growth.db` | IG per-post analytics + IG account daily snapshots + X/Twitter per-post analytics + IG/TikTok profile growth |

### Scripts

| Script | Purpose | Schedule |
|--------|---------|----------|
| collect.py | YouTube analytics collection | Daily 1:30am PST |
| ig_collect.py | Instagram per-post + account snapshots | Daily 1am PST |
| x_collect.py | X/Twitter per-post analytics | Daily 1:15am PST |
| social_scrape.py | IG/TikTok profile growth scraping | Daily 1:45am PST |
| query.py | YouTube analytics CLI | On demand |
| ig_query.py | Instagram analytics CLI | On demand |
| x_query.py | X/Twitter analytics CLI | On demand |
| charts.py | Generate PNG charts (dark theme) | On demand |

### Metrics by Platform

**YouTube:** Per-video views, watch time, engagement, subscriber conversion (which videos drive new subs vs just views)

**Instagram:** Per-post engagement (views, reach, likes, comments, saves, shares, follows, reel view time), account growth (followers, following, media count, profile views, website clicks, reach, accounts engaged)

**X/Twitter:** Per-post impressions, likes, retweets, bookmarks, shares, follower gains (X API v2), dual view tracking (public_metrics + analytics endpoint)

**TikTok:** Follower growth

### Cron Jobs

| Job | Schedule |
|-----|----------|
| YouTube Collect | Daily 1:30am PST |
| Instagram Collect | Daily 1am PST |
| X Collect | Daily 1:15am PST |
| IG/TikTok Scrape | Daily 1:45am PST |

## Steps

### 1. Create Social Tracker Directory

```
cmd --session <id> --command "mkdir -p ~/clawd/tools/social-tracker/db ~/clawd/tools/social-tracker/scripts"
```

### 2. Install Python Dependencies

```
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && pip install -r requirements.txt"
```

### 3. Configure YouTube OAuth via gog

If not already done during Google Workspace setup, authenticate with YouTube scopes.

```
cmd --session <id> --command "gog auth login --scopes youtube-data,youtube-analytics"
```

### 4. Configure Instagram Token

Set the Meta Graph API long-lived token. This token must be refreshed every 60 days.

```
cmd --session <id> --command "openclaw config set social.instagram-token '<INSTAGRAM_ACCESS_TOKEN>'"
```

To refresh an expiring token:

```
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 ig_collect.py --refresh"
```

This updates the token in `~/.openclaw/.env` automatically.

### 5. Configure X/Twitter OAuth 1.0a Keys

Set all four OAuth 1.0a keys in the environment.

```
cmd --session <id> --command "openclaw config set social.x-api-key '<X_API_KEY>'"
cmd --session <id> --command "openclaw config set social.x-api-secret '<X_API_SECRET>'"
cmd --session <id> --command "openclaw config set social.x-access-token '<X_ACCESS_TOKEN>'"
cmd --session <id> --command "openclaw config set social.x-access-token-secret '<X_ACCESS_TOKEN_SECRET>'"
```

### 6. Run Initial Collection

Run each collector manually to verify authentication and populate the databases.

```
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 collect.py"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 ig_collect.py"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 x_collect.py"
```

### 7. Set Up Cron Jobs (Staggered 1am-1:45am PST)

Add all four collection jobs, staggered to avoid API rate limits.

```
cmd --session <id> --command "openclaw cron add --name 'Instagram Collect' --schedule '0 1 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/social-tracker && python3 ig_collect.py'"
cmd --session <id> --command "openclaw cron add --name 'X Collect' --schedule '15 1 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/social-tracker && python3 x_collect.py'"
cmd --session <id> --command "openclaw cron add --name 'YouTube Collect' --schedule '30 1 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/social-tracker && python3 collect.py'"
cmd --session <id> --command "openclaw cron add --name 'IG/TikTok Scrape' --schedule '45 1 * * *' --tz 'America/Los_Angeles' --command 'cd ~/clawd/tools/social-tracker && python3 social_scrape.py'"
```

### 8. Test Queries

Verify the query scripts can read from the populated databases.

```
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 query.py summary"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 ig_query.py summary"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 x_query.py summary"
```

## Verification

Run these commands to confirm the social tracker is fully operational:

```
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 query.py summary"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 ig_query.py top --metric views -n 5"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 x_query.py summary"
cmd --session <id> --command "cd ~/clawd/tools/social-tracker && python3 charts.py"
```

Expected output:
- YouTube summary should show total videos tracked, recent views, subscriber changes
- Instagram top posts should list top 5 by views with engagement metrics
- X/Twitter summary should show recent post performance and follower trend
- Charts should generate PNG files in the output directory (dark theme)

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| YouTube auth failure | OAuth token expired or wrong scopes | Re-run `gog auth login` and ensure both YouTube Data API and Analytics API scopes are included. |
| Instagram token expired | Long-lived token past 60-day expiry | Run `python3 ig_collect.py --refresh` to exchange for a new token. Updates .env automatically. |
| X/Twitter rate limits | Too many API calls in window | Built-in 350ms delay between calls. For catch-up after downtime, use `--backfill N` flag to gradually backfill N days. |
| Empty data in databases | API keys missing or incorrect | Check all API keys in `~/.openclaw/.env`. Run individual collectors with `--verbose` flag for detailed error output. |
| Charts fail to generate | matplotlib or dependencies missing | Run `pip install matplotlib` and verify Python 3 is the default python3. |
| TikTok scraping fails | Website structure changed | TikTok scraping is fragile. Check if social_scrape.py selectors need updating. Follower count may need manual entry as fallback. |

## Dependencies

- **Requires:** `deploy-messaging-setup.md`, `deploy-google-workspace.md` (for YouTube)
- **Required by:** `deploy-daily-briefing.md`, `deploy-advisory-council.md`
