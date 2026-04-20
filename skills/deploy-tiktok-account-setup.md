---
name: "Deploy TikTok Account Setup"
category: "integration"
subcategory: "social-posting"
third_party_service: "TikTok"
auth_type: "scraping"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "tiktok-uploader"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "TikTok account (manually created)"
  - "TikTok sessionid cookie"
  - "Content pipeline database (deploy-content-pipeline.md)"
docs_urls:
  - "https://github.com/wkaisertexas/tiktok-uploader"
---

# Deploy TikTok Account Setup

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)

## Purpose

Set up a TikTok account for automated content posting. The human creates the account manually (phone app or browser — 5 minutes), then provides the `sessionid` cookie. The operator builds the cookies file and records account details in the content pipeline database.

**Why manual creation:** TikTok's anti-bot detection (device fingerprinting, behavioral analysis, CAPTCHAs) makes automated account creation fragile and high-maintenance. Manual creation is 100% reliable. Once the account exists, everything else is automated.

**When to use:** When setting up a new TikTok account for content posting. Run this BEFORE `deploy-tiktok-warmup.md`.

**What this skill does:**
1. Guides the human through account creation (phone or browser)
2. Collects the `sessionid` cookie value
3. Builds the cookies file for tiktok-uploader and warm-up scripts
4. Records account details in the content pipeline database
5. Validates the session is working

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CONTENT_DB}` | Content pipeline database | `${WORKSPACE}/data/content-pipeline.db` |
| `${TIKTOK_SESSION_ID}` | Human provides: from Chrome DevTools | `a1b2c3d4e5f6g7h8...` |
| `${TIKTOK_USERNAME}` | Human provides: the account username | `@relcontent` |
| `${TIKTOK_COOKIES_PATH}` | Output: generated cookies file | `${WORKSPACE}/config/tiktok-cookies.txt` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks:**

1. Content pipeline database exists:
   ```
   sqlite3 ${CONTENT_DB} "SELECT value FROM meta WHERE key='schema_version'" 2>/dev/null || echo 'MISSING'
   ```
   Expected: `1.0.0`. If missing, deploy `deploy-content-pipeline.md` first.

2. Config directory exists:
   ```
   test -d ${WORKSPACE}/config && echo 'EXISTS' || echo 'MISSING'
   ```
   If missing, create it.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Cookie format | Netscape cookies.txt | tiktok-uploader also supports JSON cookie list |
| Cookie path | `${WORKSPACE}/config/tiktok-cookies.txt` | Client wants cookies elsewhere |
| Cookie refresh | Manual (every ~2 months) | Set up a Telegram reminder for proactive refresh |

## Prerequisites

- `deploy-content-pipeline.md` deployed
- Human has created a TikTok account (phone app or browser)
- Human has logged into TikTok in Chrome on desktop

## Steps

### Phase 1: Human Creates Account `[HUMAN_INPUT]`

#### 1.1 Guide the human through account creation

Tell the human:

> **Create a TikTok account (5 minutes):**
>
> 1. Download TikTok on your phone OR go to tiktok.com in a browser
> 2. Sign up with email or phone number
> 3. Choose **Personal/Creator account** (NOT Business — creator accounts get 3x better engagement)
> 4. Pick a username that signals your niche
> 5. Set a profile picture and bio (can be changed later)
>
> **After creating the account:**
>
> 1. Log into the SAME account in **Chrome on your computer** at tiktok.com
> 2. Press **F12** to open DevTools
> 3. Go to **Application** tab → **Cookies** → **https://www.tiktok.com**
> 4. Find the row named **`sessionid`**
> 5. Copy the **Value** (long string of letters and numbers)
> 6. Send me that value
>
> **Important:** Do NOT log out of Chrome after copying. Logging out invalidates the session.

Wait for the human to provide the `sessionid` value and optionally the username.

### Phase 2: Build Cookies File

#### 2.1 Generate the cookies file from sessionid `[AUTO]`

Build a Netscape-format cookies.txt from the provided sessionid. This format is what tiktok-uploader expects.

**Remote:**

Write to `${TIKTOK_COOKIES_PATH}`:

```
# Netscape HTTP Cookie File
.tiktok.com	TRUE	/	TRUE	2147483647	sessionid	${TIKTOK_SESSION_ID}
.tiktok.com	TRUE	/	TRUE	2147483647	sessionid_ss	${TIKTOK_SESSION_ID}
```

Note: Use the base64 file transfer method from `_deploy-common.md` since the session ID may contain special characters.

Expected: File exists at `${TIKTOK_COOKIES_PATH}` with the sessionid value.

#### 2.2 Verify the session is valid `[AUTO]`

Test that the cookies work by making a lightweight request to TikTok's API.

**Remote:**
```
python3 -c "
from playwright.sync_api import sync_playwright
import sys

cookies_path = '${TIKTOK_COOKIES_PATH}'
with open(cookies_path) as f:
    lines = [l.strip() for l in f if l.strip() and not l.startswith('#')]

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context()
    for line in lines:
        parts = line.split('\t')
        if len(parts) >= 7:
            context.add_cookies([{
                'name': parts[5],
                'value': parts[6],
                'domain': '.tiktok.com',
                'path': '/',
            }])
    page = context.new_page()
    page.goto('https://www.tiktok.com/foryou', timeout=15000)
    page.wait_for_timeout(3000)
    # If logged in, the page title or URL won't redirect to login
    url = page.url
    logged_in = 'login' not in url.lower()
    browser.close()
    print('SESSION_VALID' if logged_in else 'SESSION_INVALID')
    sys.exit(0 if logged_in else 1)
"
```

Expected: `SESSION_VALID`.

If this fails: The sessionid is invalid or expired. Ask the human to re-copy it from DevTools, ensuring they copied the full value and haven't logged out.

### Phase 3: Record Account Details

#### 3.1 Save account info to database `[AUTO]`

Record the TikTok account details in the content pipeline database.

**Remote:**
```
sqlite3 ${CONTENT_DB} "
INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('tiktok_username', '${TIKTOK_USERNAME}', datetime('now'));
INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('tiktok_cookies_path', '${TIKTOK_COOKIES_PATH}', datetime('now'));
INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('tiktok_setup_date', datetime('now'), datetime('now'));
INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('tiktok_cookies_expires_approx', datetime('now', '+60 days'), datetime('now'));
"
```

Expected: 4 rows inserted into meta table.

#### 3.2 Set cookie refresh reminder `[GUIDED]`

TikTok session cookies expire approximately every 2 months. Set a reminder so the agent proactively asks the human to refresh cookies before they expire.

Store the approximate expiry in the database (done in 3.1 via `tiktok_cookies_expires_approx`). The agent's daily health check or briefing should flag when cookies are within 7 days of expiry.

Optionally schedule a cron or Telegram reminder for 7 weeks from now:
```
At 7 weeks: "Your TikTok cookies expire soon. Please log into tiktok.com in Chrome, copy the new sessionid from DevTools (F12 → Application → Cookies), and send it to me."
```

## Verification

1. **Cookies file exists and has sessionid:**
   ```
   grep -c 'sessionid' ${TIKTOK_COOKIES_PATH}
   ```
   Expected: `2` (sessionid + sessionid_ss).

2. **Account recorded in database:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT key, value FROM meta WHERE key LIKE 'tiktok_%'"
   ```
   Expected: username, cookies_path, setup_date, cookies_expires_approx.

3. **Session is valid (from 2.2 above):**
   Expected: `SESSION_VALID`.

## Cookie Refresh Process

When cookies expire (~every 2 months), the human needs to:

1. Log into tiktok.com in Chrome (same account)
2. F12 → Application → Cookies → tiktok.com → copy `sessionid` value
3. Send it to the agent
4. Agent rebuilds the cookies file (repeat Phase 2)

The agent should proactively remind the human before expiry using the `tiktok_cookies_expires_approx` value in the database.

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Guide human | SKIP (needs human) | Guide | Guide |
| 2.1 Build cookies | Execute | Execute | Confirm |
| 2.2 Verify session | Execute | Execute | Confirm |
| 3.1 Save to DB | Execute | Execute | Confirm |
| 3.2 Set reminder | Execute | Confirm | Confirm |

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `SESSION_INVALID` | sessionid expired or copied incorrectly | Re-copy from DevTools; ensure not logged out |
| Empty sessionid in DevTools | Not logged in on this browser | Log in to tiktok.com first |
| Multiple sessionid entries | Normal — TikTok sets several session cookies | Copy the one named exactly `sessionid` |
| Uploads fail after working | Cookies expired (~2 months) | Refresh: re-copy sessionid from DevTools |

## Dependencies

- **Depends on:** `deploy-content-pipeline.md` (database)
- **Required by:** `deploy-tiktok-warmup.md` (cookies), `deploy-tiktok-posting.md` (cookies)
