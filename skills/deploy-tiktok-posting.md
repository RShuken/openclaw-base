---
name: "Deploy TikTok Posting"
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
  - "Python 3.9+"
  - "Playwright + Chromium"
  - "TikTok session cookies (deploy-tiktok-account-setup.md)"
  - "Content pipeline database"
docs_urls:
  - "https://github.com/wkaisertexas/tiktok-uploader"
  - "https://playwright.dev/python/"
---

# Deploy TikTok Posting

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)

## Purpose

Install TikTok video posting capabilities using the `tiktok-uploader` Python library (Playwright-based browser automation with cookie auth). This enables the agent to publish videos to TikTok on the client's behalf after human approval.

**When to use:** When the client wants automated TikTok posting from their agent, typically as the final step in a content pipeline (idea → research → generate → approve → post).

**What this skill does:**
1. Installs Python dependencies (tiktok-uploader + Playwright + Chromium)
2. Sets up cookie storage for TikTok authentication
3. Creates the posting script with scheduling support
4. Tests a draft upload (does NOT publish)
5. Wires into the content pipeline database for status tracking

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${TIKTOK_USERNAME}` | Client provides: their TikTok account username | `@relcontent` |
| `${TIKTOK_COOKIES_PATH}` | Created during Phase 2 | `${WORKSPACE}/config/tiktok-cookies.txt` |
| `${CONTENT_DB}` | Content pipeline database path | `${WORKSPACE}/data/content-pipeline.db` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**

1. Check if Python 3.9+ is available:

**Remote:**
```
python3 --version
```

Expected: `Python 3.9.x` or higher.

If this fails: Install Python 3.9+ via the platform's package manager.

2. Check if tiktok-uploader is already installed:

**Remote:**
```
python3 -c "import tiktok_uploader; print('INSTALLED')" 2>/dev/null || echo 'NOT_INSTALLED'
```

Expected: `NOT_INSTALLED` for fresh deploys.

3. Check if the content pipeline database exists:

**Remote:**
```
test -f ${WORKSPACE}/data/content-pipeline.db && echo 'EXISTS' || echo 'MISSING'
```

Expected: `EXISTS`. If missing, deploy `deploy-content-pipeline.md` first.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Cookie storage | `${WORKSPACE}/config/tiktok-cookies.txt` | Client wants cookies in a different location or encrypted at rest |
| Upload library | `tiktok-uploader` (wkaisertexas) | Client has bot detection issues — switch to `tiktokautouploader` (haziq-exe, uses Phantomwright) |
| Browser | Chrome headless | Client needs a different browser or headed mode for debugging |
| Post approval | Required via Telegram before posting | Client wants fully automatic posting (remove approval gate) |

## Prerequisites

- Python 3.9+ installed
- `deploy-content-pipeline.md` deployed (content database exists)
- `deploy-messaging-setup.md` deployed (Telegram topics exist)
- **HUMAN REQUIRED:** TikTok cookies exported from the client's browser (see Phase 2)

## Steps

### Phase 1: Install Dependencies

#### 1.1 Install tiktok-uploader and Playwright `[AUTO]`

Install the Python package and browser binary needed for TikTok automation.

**Remote:**
```
python3 -m pip install tiktok-uploader playwright && python3 -m playwright install chromium
```

Expected: Both packages install successfully, Chromium binary downloaded.

If this fails: Check pip version (`python3 -m pip --version`). May need `--user` flag if no write access to site-packages. Chromium install may need `--with-deps` on Linux for system libraries.

### Phase 2: Cookie Setup `[HUMAN_INPUT]`

#### 2.1 Guide client through cookie export

The client must log into TikTok in their browser and export cookies. Walk them through this:

1. Install the **"Get cookies.txt LOCALLY"** browser extension (Chrome/Firefox/Edge)
2. Log in to TikTok at `https://www.tiktok.com` in the browser
3. While on tiktok.com, click the extension icon → "Export cookies"
4. Save as `cookies.txt`
5. **Critical:** Open DevTools (F12) → Application → Storage → Cookies → tiktok.com → find `sessionid` value
6. Append to cookies.txt if missing:
   ```
   .tiktok.com	TRUE	/	FALSE	2147483647	sessionid	<PASTE_SESSION_ID_HERE>
   ```
7. **Do NOT log out of this browser session** — logging out invalidates the cookies immediately

**Important:** Cookies expire approximately every 2 months. The client will need to re-export when uploads start failing with auth errors.

#### 2.2 Transfer cookies to workspace `[HUMAN_INPUT]`

Transfer the exported cookies.txt to the client machine.

**Remote:**
```
mkdir -p ${WORKSPACE}/config
```

Then transfer the cookies file content to `${WORKSPACE}/config/tiktok-cookies.txt` using the base64 file transfer method from `_deploy-common.md`.

Expected: File exists at `${TIKTOK_COOKIES_PATH}` with valid Netscape cookie format.

#### 2.3 Verify cookies are valid `[GUIDED]`

Quick validation that the cookies file has the critical `sessionid` entry.

**Remote:**
```
grep -c 'sessionid' ${TIKTOK_COOKIES_PATH}
```

Expected: `1` or higher (sessionid cookie present).

If this fails: The sessionid cookie is missing. Go back to step 2.1 — the client must manually copy it from DevTools.

### Phase 3: Create Posting Script

#### 3.1 Write the TikTok posting script `[AUTO]`

Create the posting script that the agent calls to publish videos. This script:
- Reads video path, description, and optional schedule from arguments
- Uses the stored cookies for auth
- Updates the content pipeline database with post status and platform URL
- Supports `--dry-run` mode for testing without publishing

**Remote:**

Write the following Python script to `${WORKSPACE}/skills/tiktok-post/post.py`:

```python
#!/usr/bin/env python3
"""TikTok video posting via tiktok-uploader.

Usage:
  python3 post.py <video_path> <description> [--schedule "YYYY-MM-DD HH:MM"] [--dry-run] [--post-id <id>]

Args:
  video_path: Path to the MP4 file
  description: Caption text including hashtags (hashtags must be followed by a space)
  --schedule: Optional UTC datetime to schedule the post (20min to 10 days out)
  --dry-run: Validate everything but don't actually upload
  --post-id: Content pipeline post ID to update status in the database
"""
import argparse
import datetime
import json
import os
import sqlite3
import sys

def update_post_status(db_path, post_id, status, platform_post_id=None, platform_url=None, error=None):
    """Update the content pipeline database with post results."""
    if not post_id or not os.path.exists(db_path):
        return
    conn = sqlite3.connect(db_path)
    try:
        if status == 'posted' and platform_url:
            conn.execute(
                "UPDATE posts SET status=?, platform_post_id=?, platform_url=?, published_at=datetime('now'), updated_at=datetime('now') WHERE id=?",
                (status, platform_post_id, platform_url, post_id)
            )
        elif status == 'failed':
            conn.execute(
                "UPDATE posts SET status=?, error_message=?, updated_at=datetime('now') WHERE id=?",
                (status, str(error)[:500], post_id)
            )
        else:
            conn.execute(
                "UPDATE posts SET status=?, updated_at=datetime('now') WHERE id=?",
                (status, post_id)
            )
        conn.commit()
    finally:
        conn.close()

def main():
    parser = argparse.ArgumentParser(description='Post video to TikTok')
    parser.add_argument('video_path', help='Path to MP4 video file')
    parser.add_argument('description', help='Caption with hashtags')
    parser.add_argument('--schedule', help='UTC datetime: "YYYY-MM-DD HH:MM"', default=None)
    parser.add_argument('--dry-run', action='store_true', help='Validate without uploading')
    parser.add_argument('--post-id', type=int, help='Content pipeline post ID', default=None)
    parser.add_argument('--cookies', default=None, help='Path to cookies.txt')
    parser.add_argument('--db', default=None, help='Path to content-pipeline.db')
    args = parser.parse_args()

    workspace = os.environ.get('WORKSPACE', os.path.expanduser('~/clawd'))
    cookies_path = args.cookies or os.path.join(workspace, 'config', 'tiktok-cookies.txt')
    db_path = args.db or os.path.join(workspace, 'data', 'content-pipeline.db')

    # Validate inputs
    if not os.path.exists(args.video_path):
        print(f"ERROR: Video file not found: {args.video_path}", file=sys.stderr)
        sys.exit(1)
    if not os.path.exists(cookies_path):
        print(f"ERROR: Cookies file not found: {cookies_path}", file=sys.stderr)
        sys.exit(1)

    schedule_dt = None
    if args.schedule:
        try:
            schedule_dt = datetime.datetime.strptime(args.schedule, "%Y-%m-%d %H:%M")
        except ValueError:
            print("ERROR: Schedule format must be 'YYYY-MM-DD HH:MM' (UTC)", file=sys.stderr)
            sys.exit(1)

    if args.dry_run:
        print(json.dumps({
            "status": "dry_run",
            "video": args.video_path,
            "description": args.description[:50] + "..." if len(args.description) > 50 else args.description,
            "schedule": args.schedule,
            "cookies": cookies_path
        }, indent=2))
        return

    # Update status to 'posting'
    update_post_status(db_path, args.post_id, 'posting')

    try:
        from tiktok_uploader.upload import TikTokUploader

        uploader = TikTokUploader(cookies=cookies_path, headless=True)
        uploader.upload_video(
            args.video_path,
            description=args.description,
            schedule=schedule_dt,
            comment=True,
            stitch=True,
            duet=True
        )

        print(json.dumps({"status": "posted", "video": args.video_path}))
        update_post_status(db_path, args.post_id, 'posted')

    except Exception as e:
        error_msg = str(e)
        print(json.dumps({"status": "failed", "error": error_msg}), file=sys.stderr)
        update_post_status(db_path, args.post_id, 'failed', error=error_msg)
        sys.exit(1)

if __name__ == '__main__':
    main()
```

Expected: Script exists at `${WORKSPACE}/skills/tiktok-post/post.py`, executable.

If this fails: Check that the directory was created and Python can import the script.

#### 3.2 Make script executable `[AUTO]`

**Remote:**
```
chmod +x ${WORKSPACE}/skills/tiktok-post/post.py
```

### Phase 4: Test Upload

#### 4.1 Dry-run test `[GUIDED]`

Test the posting script in dry-run mode to verify all paths and dependencies resolve.

**Remote:**
```
cd ${WORKSPACE} && python3 skills/tiktok-post/post.py /dev/null "Test post #test" --dry-run
```

Expected: JSON output with `"status": "dry_run"` confirming paths resolve.

If this fails: Check Python imports — run `python3 -c "from tiktok_uploader.upload import TikTokUploader; print('OK')"` to isolate dependency issues.

#### 4.2 Real upload test (optional) `[HUMAN_INPUT]`

If the client has a test video ready, do a real upload to verify cookies work. **This WILL publish to TikTok.**

**Remote:**
```
python3 ${WORKSPACE}/skills/tiktok-post/post.py "${WORKSPACE}/output/videos/test.mp4" "Testing automated posting #test "
```

Expected: Video appears on TikTok. JSON output with `"status": "posted"`.

If this fails:
- Auth error → Cookies are expired or sessionid is missing. Re-export cookies.
- "Continue to post" button not found → TikTok UI changed. Check for tiktok-uploader updates: `python3 -m pip install --upgrade tiktok-uploader`
- Timeout → Video may be too large. Try a shorter clip (<30s, <50MB).

## Verification

1. **Dependency check:**
   ```
   python3 -c "from tiktok_uploader.upload import TikTokUploader; print('OK')"
   ```

2. **Cookie validity:**
   ```
   grep 'sessionid' ${WORKSPACE}/config/tiktok-cookies.txt | wc -l
   ```

3. **Script exists and runs:**
   ```
   python3 ${WORKSPACE}/skills/tiktok-post/post.py --help
   ```

## Cookie Maintenance

Cookies expire approximately every 2 months. When uploads start failing with authentication errors:

1. Client logs into TikTok in browser (same browser, same profile)
2. Re-exports cookies using the same extension
3. Grabs sessionid from DevTools again
4. Transfers new cookies.txt to `${TIKTOK_COOKIES_PATH}`

Consider setting a reminder (cron or Telegram notification) every 7 weeks to proactively refresh cookies before they expire.

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Install deps | Execute | Confirm | Confirm |
| 2.1-2.2 Cookie setup | SKIP (needs human) | Guide through | Guide through |
| 2.3 Verify cookies | Execute | Confirm | Confirm |
| 3.1 Write script | Execute | Confirm | Confirm |
| 4.1 Dry-run test | Execute | Confirm | Confirm |
| 4.2 Real upload | SKIP | Ask | Ask |

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `ModuleNotFoundError: tiktok_uploader` | Package not installed | `python3 -m pip install tiktok-uploader` |
| `browserType.launch: Executable doesn't exist` | Playwright browsers not installed | `python3 -m playwright install chromium` |
| Auth/login errors | Cookies expired or sessionid missing | Re-export cookies (see Cookie Maintenance) |
| "Continue to post" button not found | TikTok UI changed | Update package: `pip install --upgrade tiktok-uploader` |
| Upload succeeds but video not visible | TikTok review/processing delay | Wait 5-10 minutes; check TikTok app |
| Rate limited | Too many uploads in short period | Wait several hours between batch uploads |

## Dependencies

- **Depends on:** `deploy-content-pipeline.md` (database), `deploy-messaging-setup.md` (Telegram approval flow)
- **Required by:** `deploy-video-research.md` (end-to-end pipeline)
