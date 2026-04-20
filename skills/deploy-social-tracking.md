# Deploy Social Media Tracker

## Compatibility

- **OpenClaw Version**: 2026.4.15+ (compat header rewritten 2026-04-20 — live-verified on 4C)
- **Status**: WORKING — previous "BLOCKED" was stale. `openclaw cron` (add/list/rm — note: `rm` not `remove`) and `openclaw config set` all exist on 2026.4.15 per `audit/_baseline.md` §3/§4C.7
- **Scheduling**: staggered nightly jobs. `openclaw cron add --name social-youtube --cron "0 22 * * *"`, `--name social-instagram --cron "15 22 * * *"`, etc. (15-min offsets avoid API rate-limit collisions)
- **Config**: platform credentials go in `~/.openclaw/.env` (mode 600); workspace files (`TOOLS.md`) document what's connected
- **Model**: agent default (Codex-first); do not hardcode

## Purpose

Track daily performance snapshots across YouTube, Instagram, X/Twitter, and TikTok. Stores per-video and per-post analytics in SQLite databases. Generates charts. Feeds data into the daily briefing and business advisory council.

**When to use:** When the client has social media accounts they want to track with automated daily analytics collection.

**What this skill does:**
1. Creates the social tracker directory structure and installs Python dependencies
2. Configures authentication for each platform (YouTube OAuth, Instagram token, X/Twitter OAuth, TikTok scraping)
3. Runs initial data collection to populate databases and verify auth
4. Schedules staggered nightly cron jobs for automated collection
5. Verifies queries and chart generation work against collected data

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

Resolve these before executing. Sources are listed for each.

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${TIMEZONE}` | Client profile: timezone | `America/Los_Angeles` |
| `${INSTAGRAM_ACCESS_TOKEN}` | Client provides: Meta Graph API long-lived token | `IGQVJ...` |
| `${X_API_KEY}` | Client provides: X/Twitter OAuth 1.0a consumer key | `abc123...` |
| `${X_API_SECRET}` | Client provides: X/Twitter OAuth 1.0a consumer secret | `xyz789...` |
| `${X_ACCESS_TOKEN}` | Client provides: X/Twitter OAuth 1.0a access token | `12345-abc...` |
| `${X_ACCESS_TOKEN_SECRET}` | Client provides: X/Twitter OAuth 1.0a access token secret | `secret...` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`: read the client profile, check what exists, identify adaptations, and present a deployment plan before executing.

**Pre-flight checks for this skill:**

Check if the social tracker is already deployed:

**Remote:**
```
ls ${WORKSPACE}/tools/social-tracker/db/ 2>/dev/null || echo 'NOT_DEPLOYED'
```

Expected: Either a list of `.db` files (already deployed) or `NOT_DEPLOYED`.

Check that Python 3 is available:

**Remote:**
```
python3 --version 2>/dev/null || echo 'NO_PYTHON3'
```

Expected: `Python 3.x.x`. If `NO_PYTHON3`, install Python 3 before proceeding.

Check Google OAuth status (needed for YouTube):

**Remote:**
```
gog auth status 2>/dev/null || echo 'GOG_NOT_CONFIGURED'
```

Expected: Shows authenticated status. If not configured, run `deploy-google-workspace.md` first.

**Decision points from pre-flight:**
- Which social platforms does the client actually use? Only deploy trackers for platforms they're on.
- Are the required API tokens available for each platform?
- Is the social tracker already partially deployed? If so, identify what's missing.

## Adaptation Points

The defaults below work for most installs. Cross-reference against the client profile for overrides.

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Platforms tracked | YouTube + Instagram + X/Twitter + TikTok | Client only uses some platforms — deploy only what's needed |
| YouTube auth | Google OAuth via `gog` | Client has limited Google scopes (fall back to YouTube Data API key only — less data) |
| Instagram auth | Meta Graph API long-lived token | Client doesn't use Instagram or can't get API access — skip |
| X/Twitter auth | OAuth 1.0a (4 tokens) | Client has limited X API access (fall back to app-only bearer token — less data), or skip |
| TikTok tracking | Web scraping (no API needed) | Client doesn't use TikTok — skip |
| Collection schedule | 1:00-1:45am PST staggered | Client's timezone differs, or different times preferred |
| Chart generation | Dark theme PNGs | Client prefers light theme, or skip charts entirely |
| Database path | `${WORKSPACE}/tools/social-tracker/db/` | Client wants databases in a different location |

## Prerequisites

- OpenClaw installed and running (`openclaw/install.md`)
- `deploy-messaging-setup.md` completed
- `deploy-google-workspace.md` completed (for YouTube OAuth via `gog`)
- Python 3 available on the client machine
- YouTube: Google OAuth token (`gog auth login` for YouTube Data API + Analytics API)
- Instagram: Meta Graph API long-lived token (refreshed every 60 days)
- X/Twitter: OAuth 1.0a tokens (4 keys from X Developer Portal)
- TikTok: No API needed (web scraping for follower count)

## What Gets Installed

### Databases

| Database | Location | Purpose |
|----------|----------|---------|
| YouTube Analytics | `${WORKSPACE}/tools/social-tracker/db/views.db` | Per-video views, watch time, avg view duration/percentage, likes, dislikes, comments, shares, subs gained/lost, thumbnail impressions/CTR + video metadata |
| Social Growth | `${WORKSPACE}/tools/social-tracker/db/social_growth.db` | IG per-post analytics + IG account daily snapshots + X/Twitter per-post analytics + IG/TikTok profile growth |

### Scripts

| Script | Purpose | Schedule |
|--------|---------|----------|
| `collect.py` | YouTube analytics collection | Daily 1:30am (client TZ) |
| `ig_collect.py` | Instagram per-post + account snapshots | Daily 1:00am (client TZ) |
| `x_collect.py` | X/Twitter per-post analytics | Daily 1:15am (client TZ) |
| `social_scrape.py` | IG/TikTok profile growth scraping | Daily 1:45am (client TZ) |
| `query.py` | YouTube analytics CLI | On demand |
| `ig_query.py` | Instagram analytics CLI | On demand |
| `x_query.py` | X/Twitter analytics CLI | On demand |
| `charts.py` | Generate PNG charts (dark theme) | On demand |

### Metrics by Platform

**YouTube:** Per-video views, watch time, engagement, subscriber conversion (which videos drive new subs vs just views)

**Instagram:** Per-post engagement (views, reach, likes, comments, saves, shares, follows, reel view time), account growth (followers, following, media count, profile views, website clicks, reach, accounts engaged)

**X/Twitter:** Per-post impressions, likes, retweets, bookmarks, shares, follower gains (X API v2), dual view tracking (public_metrics + analytics endpoint)

**TikTok:** Follower growth (web scraping)

## Steps

### Phase 1: Directory Structure and Dependencies

#### 1.1 Create Social Tracker Directory Tree `[AUTO]`

Create the directory structure for scripts, databases, and output.

**Remote:**
```
mkdir -p ${WORKSPACE}/tools/social-tracker/db ${WORKSPACE}/tools/social-tracker/scripts ${WORKSPACE}/tools/social-tracker/output
```

Expected: Directories created without errors.

If already exists: Safe to re-run. `mkdir -p` is idempotent.

#### 1.2 Install Python Dependencies `[AUTO]`

Install the Python packages needed by the collection and charting scripts.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 -m pip install -r requirements.txt
```

Expected: All packages install successfully. Key dependencies include `google-api-python-client`, `requests`, `matplotlib`, `beautifulsoup4`.

If this fails: Check that Python 3 is installed (`python3 --version`). If `requirements.txt` doesn't exist yet, the scripts need to be written first (they're created as part of the OpenClaw workspace scaffolding). If pip itself is missing, install it: `python3 -m ensurepip --upgrade`.

If already installed: Safe to re-run. `python3 -m pip install` is idempotent for already-satisfied requirements.

### Phase 2: Platform Authentication `[HUMAN_INPUT]`

Each platform requires its own authentication. Only configure platforms the client uses. Confirm with the client which platforms to set up.

#### 2.1 Configure YouTube OAuth via gog

If not already done during Google Workspace setup, authenticate with YouTube scopes. This requires the client to complete an OAuth flow.

**Remote:**
```
gog auth login --scopes youtube-data,youtube-analytics
```

Expected: OAuth flow completes and tokens are stored for YouTube Data API + Analytics API access.

If this fails: Ensure `deploy-google-workspace.md` has been run and `gog` is installed. The client may need to enable the YouTube Data API and YouTube Analytics API in their Google Cloud Console.

If already configured: Check with `gog auth status`. If YouTube scopes are already present, skip this step.

#### 2.2 Configure Instagram Token `[HUMAN_INPUT]`

Set the Meta Graph API long-lived token. The client must provide this token from the Meta Developer Portal. This token expires every 60 days and must be refreshed.

**Remote:**
```
openclaw config set social.instagram-token '${INSTAGRAM_ACCESS_TOKEN}'
```

Expected: Token stored in OpenClaw config.

To refresh an expiring token (can be done later):

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 ig_collect.py --refresh
```

Expected: Token refreshed and updated in `~/.openclaw/.env` automatically.

If this fails: The token may already be expired. The client needs to generate a new long-lived token from the Meta Developer Portal.

#### 2.3 Configure X/Twitter OAuth 1.0a Keys `[HUMAN_INPUT]`

Set all four OAuth 1.0a keys. The client must provide these from the X Developer Portal.

**Remote:**
```
openclaw config set social.x-api-key '${X_API_KEY}'
```
```
openclaw config set social.x-api-secret '${X_API_SECRET}'
```
```
openclaw config set social.x-access-token '${X_ACCESS_TOKEN}'
```
```
openclaw config set social.x-access-token-secret '${X_ACCESS_TOKEN_SECRET}'
```

Expected: All four keys stored in OpenClaw config.

If this fails: Verify the client has an approved X Developer account with OAuth 1.0a access. App-only bearer tokens provide less data but can be used as a fallback.

#### 2.4 Verify TikTok Scraping Access `[AUTO]`

TikTok tracking uses web scraping (no API needed), but verify the client's TikTok profile URL is accessible.

**Remote:**
```
python3 -c "import requests; r = requests.get('https://www.tiktok.com/@CLIENT_HANDLE', headers={'User-Agent':'Mozilla/5.0'}); print(f'Status: {r.status_code}')"
```

Expected: `Status: 200`. Replace `CLIENT_HANDLE` with the client's actual TikTok username.

If this fails: TikTok may be blocked by the client's network, or the profile URL is wrong. TikTok scraping is fragile by nature — follower count may need manual entry as a fallback.

### Phase 3: Initial Data Collection

#### 3.1 Run YouTube Collection `[GUIDED]`

Run the YouTube collector manually to verify authentication and populate the database with initial data.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 collect.py
```

Expected: Collection completes without auth errors. Creates/populates `${WORKSPACE}/tools/social-tracker/db/views.db` with per-video analytics data.

If this fails: Check YouTube OAuth with `gog auth status`. Common issues: expired token (re-run `gog auth login`), missing API scopes, or YouTube Analytics API not enabled in Google Cloud Console.

#### 3.2 Run Instagram Collection `[GUIDED]`

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 ig_collect.py
```

Expected: Collection completes. Populates `${WORKSPACE}/tools/social-tracker/db/social_growth.db` with per-post and account snapshot data.

If this fails: Token may be expired or invalid. Run `python3 ig_collect.py --verbose` for detailed error output. If the token is expired, refresh it (see step 2.2).

#### 3.3 Run X/Twitter Collection `[GUIDED]`

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 x_collect.py
```

Expected: Collection completes. Populates X/Twitter post analytics in `social_growth.db`.

If this fails: Check all four OAuth keys are set correctly. X API has rate limits — there's a built-in 350ms delay between calls. For catch-up after downtime, use `--backfill N` to gradually backfill N days.

#### 3.4 Run TikTok/IG Profile Scrape `[GUIDED]`

Run the social scraper to collect TikTok follower count and Instagram profile growth data.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 social_scrape.py
```

Expected: Scrape completes. TikTok follower count and IG profile metrics written to `social_growth.db`.

If this fails: TikTok scraping depends on website structure which changes periodically. Check if `social_scrape.py` selectors need updating. Instagram profile scraping may also fail if the token is invalid. Follower count can be entered manually as a fallback.

### Phase 4: Schedule Automated Collection

#### 4.1 Add Staggered Cron Jobs `[GUIDED]`

Schedule all four collection jobs, staggered to avoid API rate limits. Adjust the timezone to the client's local timezone.

Instagram collection at 1:00am:

**Remote:**
```
openclaw cron add --name 'Instagram Collect' --schedule '0 1 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/social-tracker && python3 ig_collect.py'
```

X/Twitter collection at 1:15am:

**Remote:**
```
openclaw cron add --name 'X Collect' --schedule '15 1 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/social-tracker && python3 x_collect.py'
```

YouTube collection at 1:30am:

**Remote:**
```
openclaw cron add --name 'YouTube Collect' --schedule '30 1 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/social-tracker && python3 collect.py'
```

IG/TikTok profile scrape at 1:45am:

**Remote:**
```
openclaw cron add --name 'IG/TikTok Scrape' --schedule '45 1 * * *' --tz '${TIMEZONE}' --command 'cd ${WORKSPACE}/tools/social-tracker && python3 social_scrape.py'
```

Expected: Four cron jobs created. Verify with `openclaw cron list`.

If this fails: Check that `openclaw cron` is available (requires OpenClaw to be installed and running). Verify the timezone string is valid (e.g., `America/Los_Angeles`, `America/New_York`, `Europe/London`).

If already exists: Check for duplicate cron jobs with `openclaw cron list`. If duplicates exist, remove the old ones before adding new ones.

### Phase 5: Verify Queries and Charts

#### 5.1 Test YouTube Queries `[AUTO]`

Verify the YouTube query script can read from the populated database.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 query.py summary
```

Expected: Shows total videos tracked, recent views, and subscriber changes.

If this fails: Check that `views.db` was populated in Phase 3. Run `ls -la ${WORKSPACE}/tools/social-tracker/db/views.db` to confirm the database exists and has data.

#### 5.2 Test Instagram Queries `[AUTO]`

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 ig_query.py summary
```

Expected: Shows recent post performance and account growth metrics.

If this fails: Check that `social_growth.db` was populated. The Instagram collector may have silently failed — re-run `python3 ig_collect.py --verbose`.

#### 5.3 Test X/Twitter Queries `[AUTO]`

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 x_query.py summary
```

Expected: Shows recent post performance and follower trend.

#### 5.4 Generate Charts `[AUTO]`

Generate analytics charts to verify the charting pipeline works.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 charts.py
```

Expected: PNG chart files generated in the output directory (dark theme by default).

If this fails: Likely missing `matplotlib`. Install it: `python3 -m pip install matplotlib`. Also verify Python 3 is the default `python3`.

## Verification

Run these checks to confirm the social tracker is fully operational:

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 query.py summary
```

Expected: YouTube summary with total videos tracked, recent views, subscriber changes.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 ig_query.py top --metric views -n 5
```

Expected: Top 5 Instagram posts by views with engagement metrics.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 x_query.py summary
```

Expected: X/Twitter summary with recent post performance and follower trend.

**Remote:**
```
cd ${WORKSPACE}/tools/social-tracker && python3 charts.py
```

Expected: PNG chart files generated in the output directory (dark theme).

**Remote:**
```
openclaw cron list
```

Expected: Four social tracking cron jobs listed with correct schedules and timezone.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| YouTube auth failure | OAuth token expired or wrong scopes | Re-run `gog auth login` with `--scopes youtube-data,youtube-analytics` |
| Instagram token expired | Long-lived token past 60-day expiry | Run `python3 ig_collect.py --refresh` to exchange for a new token (updates .env automatically) |
| X/Twitter rate limits | Too many API calls in window | Built-in 350ms delay between calls. For catch-up after downtime, use `--backfill N` flag |
| Empty data in databases | API keys missing or incorrect | Check all API keys in `~/.openclaw/.env`. Run individual collectors with `--verbose` flag |
| Charts fail to generate | matplotlib or dependencies missing | Run `python3 -m pip install matplotlib` and verify Python 3 is available |
| TikTok scraping fails | Website structure changed | Check if `social_scrape.py` selectors need updating. Follower count may need manual entry as fallback |
| `python3 -m pip` not found | pip not installed for Python 3 | Run `python3 -m ensurepip --upgrade` or install pip via the OS package manager |
| Duplicate cron jobs | Skill re-run without cleanup | List with `openclaw cron list`, remove duplicates with `openclaw cron rm --name '...'` |

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| Phase 1: Directory + Dependencies | Execute silently | Confirm before starting phase | Confirm each command |
| Phase 2: Platform Auth | Pause — requires client to provide API keys/tokens | Pause — requires client input | Pause — requires client input |
| Phase 3: Initial Collection | Execute silently | Confirm before each collector | Confirm each command |
| Phase 4: Schedule Cron Jobs | Execute silently | Confirm schedule and timezone before creating | Confirm each cron job |
| Phase 5: Verify Queries | Execute silently | Execute silently | Show output, confirm each |

## Dependencies

- **Depends on:** `openclaw/install.md`, `deploy-messaging-setup.md`, `deploy-google-workspace.md` (for YouTube OAuth)
- **Required by:** `deploy-daily-briefing.md`, `deploy-advisory-council.md`
- **Related:** Chart output can be sent via Telegram topics (see `deploy-messaging-setup.md`)
