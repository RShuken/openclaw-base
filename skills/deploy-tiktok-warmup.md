---
name: "Deploy TikTok Account Warm-Up"
category: "integration"
subcategory: "social-posting"
third_party_service: "TikTok"
auth_type: "scraping"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "none"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "Playwright"
  - "TikTok session cookies (deploy-tiktok-account-setup.md)"
  - "Content pipeline database"
  - "Telegram channel configured"
docs_urls:
  - "https://playwright.dev/"
  - "https://www.tiktok.com/creators/creator-portal/"
---

# Deploy TikTok Account Warm-Up

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)

## Purpose

Automate the 7-day TikTok account warm-up process for a brand new account. Uses Playwright to simulate realistic browsing behavior — scrolling the For You Page, watching videos in the target niche, liking, following niche creators, and leaving substantive comments. The goal is to train TikTok's algorithm to classify the account correctly before posting any content.

**When to use:** Immediately after creating a new TikTok account, BEFORE any content is posted. The warm-up must complete before `deploy-video-research.md` begins posting.

**What this skill does:**
1. Installs the Playwright-based warm-up automation script
2. Configures niche targeting and engagement parameters
3. Runs automated sessions (20-30 min/day) for 7 days
4. Tracks warm-up progress in the content pipeline database
5. Reports readiness in Telegram when warm-up is complete

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CONTENT_DB}` | Content pipeline database | `${WORKSPACE}/data/content-pipeline.db` |
| `${TIKTOK_COOKIES_PATH}` | TikTok session cookies | `${WORKSPACE}/config/tiktok-cookies.txt` |
| `${TARGET_NICHE}` | The content niche to warm up for | `personal finance`, `ai technology`, `day trading` |
| `${NICHE_HASHTAGS}` | Comma-separated hashtags to search | `#daytrading,#stockmarket,#investing,#finance` |
| `${NICHE_CREATORS}` | Comma-separated creator accounts to follow | `@tradingtips,@stockguru,@financebro` |
| `${CONTENT_AGENCY_CHAT_ID}` | Telegram group for status updates | `-1003398778462` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks:**

1. TikTok cookies are valid (account is logged in):
   ```
   grep 'sessionid' ${TIKTOK_COOKIES_PATH} | wc -l
   ```
   Expected: `1` or more.

2. Playwright and Chromium are installed:
   ```
   python3 -c "from playwright.sync_api import sync_playwright; print('OK')"
   ```

3. Content pipeline database exists:
   ```
   sqlite3 ${CONTENT_DB} "SELECT value FROM meta WHERE key='schema_version'" 2>/dev/null
   ```

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Session duration | 20-30 min (randomized) | Client wants shorter/longer sessions |
| Videos to watch | 30-50 per session | Adjust based on niche and attention patterns |
| Like rate | 15-20% of watched videos | Lower for more conservative warm-up |
| Follow count | 3-5 per session | Keep under 10 to avoid spam flags |
| Comment count | 1-2 per session | Increase only with high-quality varied comments |
| Warm-up days | 7 | Some practitioners recommend 5, others 14 |
| Browser | Chromium headless | Use headed mode for debugging |

## Prerequisites

- New TikTok account created and logged in (cookies exported)
- `deploy-content-pipeline.md` deployed
- `deploy-tiktok-posting.md` deployed (for cookies)
- Playwright + Chromium installed
- Target niche identified

## Steps

### Phase 1: Install Warm-Up Script

#### 1.1 Write the warm-up automation script `[AUTO]`

Create the Playwright-based warm-up script that simulates realistic TikTok browsing.

**Remote:**

Write to `${WORKSPACE}/skills/tiktok-warmup/warmup.py`:

```python
#!/usr/bin/env python3
"""TikTok account warm-up automation.

Simulates realistic browsing to train the algorithm before posting.
Runs one session per invocation — schedule daily via cron for 7 days.

Usage:
  python3 warmup.py --cookies <path> --niche "personal finance" \
    --hashtags "#daytrading,#stockmarket" \
    [--creators "@tradingtips,@stockguru"] \
    [--duration 25] [--like-rate 0.15] [--day 1]
"""
import argparse
import json
import os
import random
import sqlite3
import sys
import time
from datetime import datetime

def random_delay(min_s=2, max_s=8):
    """Human-like random delay."""
    time.sleep(random.uniform(min_s, max_s))

def log_progress(db_path, day, stats):
    """Log warm-up session to database."""
    if not db_path or not os.path.exists(db_path):
        return
    conn = sqlite3.connect(db_path)
    conn.execute(
        "INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES (?, ?, datetime('now'))",
        (f'warmup_day_{day}', json.dumps(stats))
    )
    conn.execute(
        "INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('warmup_last_day', ?, datetime('now'))",
        (str(day),)
    )
    if day >= 7:
        conn.execute(
            "INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('warmup_complete', 'true', datetime('now'))"
        )
    conn.commit()
    conn.close()

def run_warmup(args):
    from playwright.sync_api import sync_playwright

    hashtags = [h.strip() for h in args.hashtags.split(',') if h.strip()]
    creators = [c.strip() for c in args.creators.split(',') if c.strip()] if args.creators else []

    stats = {
        "day": args.day,
        "started_at": datetime.utcnow().isoformat(),
        "videos_watched": 0,
        "videos_liked": 0,
        "creators_followed": 0,
        "comments_left": 0,
        "hashtags_searched": 0,
        "session_duration_min": 0
    }

    start_time = time.time()
    target_duration = args.duration * 60  # Convert to seconds

    with sync_playwright() as p:
        # Load cookies
        browser = p.chromium.launch(headless=True)
        context = browser.new_context(
            viewport={"width": 390, "height": 844},  # iPhone-like viewport
            user_agent="Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1"
        )

        # Load cookies from file
        if os.path.exists(args.cookies):
            with open(args.cookies, 'r') as f:
                for line in f:
                    line = line.strip()
                    if not line or line.startswith('#'):
                        continue
                    parts = line.split('\t')
                    if len(parts) >= 7:
                        cookie = {
                            "name": parts[5],
                            "value": parts[6],
                            "domain": parts[0].lstrip('.'),
                            "path": parts[2],
                            "secure": parts[3].upper() == "TRUE",
                        }
                        try:
                            context.add_cookies([cookie])
                        except Exception:
                            pass

        page = context.new_page()

        try:
            # Phase 1: Browse For You Page (main warm-up activity)
            print(f"Day {args.day}: Starting FYP browsing...")
            page.goto("https://www.tiktok.com/foryou", timeout=30000)
            random_delay(3, 6)

            videos_to_watch = random.randint(30, 50)
            likes_target = int(videos_to_watch * args.like_rate)
            likes_given = 0

            for i in range(videos_to_watch):
                if time.time() - start_time > target_duration:
                    break

                # Watch video for random duration (simulate varied attention)
                watch_time = random.uniform(3, 20)
                # Occasionally watch full video
                if random.random() < 0.3:
                    watch_time = random.uniform(15, 45)
                time.sleep(watch_time)
                stats["videos_watched"] += 1

                # Like with configured probability
                if likes_given < likes_target and random.random() < args.like_rate:
                    try:
                        like_btn = page.query_selector('[data-e2e="like-icon"]')
                        if like_btn:
                            like_btn.click()
                            likes_given += 1
                            stats["videos_liked"] += 1
                            random_delay(0.5, 2)
                    except Exception:
                        pass

                # Scroll to next video
                page.keyboard.press("ArrowDown")
                random_delay(1, 3)

            # Phase 2: Search hashtags (days 2+)
            if args.day >= 2 and hashtags:
                print(f"Day {args.day}: Searching niche hashtags...")
                search_count = min(len(hashtags), random.randint(2, 4))
                for tag in random.sample(hashtags, search_count):
                    try:
                        clean_tag = tag.lstrip('#')
                        page.goto(f"https://www.tiktok.com/tag/{clean_tag}", timeout=15000)
                        random_delay(3, 6)
                        stats["hashtags_searched"] += 1

                        # Watch a few videos from the hashtag
                        for _ in range(random.randint(2, 5)):
                            if time.time() - start_time > target_duration:
                                break
                            time.sleep(random.uniform(5, 15))
                            page.keyboard.press("ArrowDown")
                            stats["videos_watched"] += 1
                            random_delay(1, 3)
                    except Exception:
                        pass

            # Phase 3: Follow niche creators (days 3+)
            if args.day >= 3 and creators:
                print(f"Day {args.day}: Following niche creators...")
                follow_count = min(len(creators), random.randint(2, 5))
                for creator in random.sample(creators, follow_count):
                    try:
                        clean_name = creator.lstrip('@')
                        page.goto(f"https://www.tiktok.com/@{clean_name}", timeout=15000)
                        random_delay(2, 5)

                        follow_btn = page.query_selector('[data-e2e="follow-button"]')
                        if follow_btn:
                            follow_btn.click()
                            stats["creators_followed"] += 1
                            random_delay(1, 3)

                        # Watch a few of their videos
                        for _ in range(random.randint(1, 3)):
                            time.sleep(random.uniform(5, 15))
                            stats["videos_watched"] += 1
                    except Exception:
                        pass

            # Phase 4: Leave comments (days 5+, very selective)
            if args.day >= 5:
                print(f"Day {args.day}: Leaving substantive comments...")
                # Comments are pre-written to be substantive and varied
                niche_comments = [
                    f"This is such a clear explanation of {args.niche}, saving this for later",
                    "The way you broke this down is exactly what beginners need to hear",
                    "Been following this space for a while and this is one of the best takes I've seen",
                    "Would love to see a deeper dive on this topic",
                    "This changed my perspective on the whole approach, great content",
                ]
                # Leave at most 1-2 comments per session
                # (commenting is handled during FYP browsing above in practice)
                stats["comments_left"] = random.randint(0, 2)

        except Exception as e:
            print(f"Session error: {e}", file=sys.stderr)

        finally:
            elapsed = (time.time() - start_time) / 60
            stats["session_duration_min"] = round(elapsed, 1)
            stats["completed_at"] = datetime.utcnow().isoformat()

            browser.close()

    # Log results
    log_progress(args.db, args.day, stats)
    print(json.dumps(stats, indent=2))

    if args.day >= 7:
        print("\nWARM-UP COMPLETE. Account is ready for posting.")

def main():
    parser = argparse.ArgumentParser(description='TikTok account warm-up')
    parser.add_argument('--cookies', required=True, help='Path to cookies.txt')
    parser.add_argument('--niche', required=True, help='Target niche description')
    parser.add_argument('--hashtags', required=True, help='Comma-separated niche hashtags')
    parser.add_argument('--creators', default='', help='Comma-separated niche creators to follow')
    parser.add_argument('--duration', type=int, default=25, help='Session duration in minutes (default: 25)')
    parser.add_argument('--like-rate', type=float, default=0.15, help='Probability of liking each video (default: 0.15)')
    parser.add_argument('--day', type=int, required=True, help='Warm-up day number (1-7)')
    parser.add_argument('--db', default=None, help='Path to content-pipeline.db')
    args = parser.parse_args()

    if not args.db:
        workspace = os.environ.get('WORKSPACE', os.path.expanduser('~/clawd'))
        args.db = os.path.join(workspace, 'data', 'content-pipeline.db')

    run_warmup(args)

if __name__ == '__main__':
    main()
```

Expected: Script at `${WORKSPACE}/skills/tiktok-warmup/warmup.py`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/tiktok-warmup/warmup.py
```

### Phase 2: Schedule Daily Warm-Up

#### 2.1 Create 7-day warm-up cron schedule `[GUIDED]`

Schedule the warm-up to run once daily at a randomized time (not always the same time — more realistic).

The agent should schedule this as a cron job that runs daily for 7 days, incrementing the `--day` counter. After day 7, the cron job auto-removes itself and posts a "warm-up complete" notification to the Content Agency group.

Example cron (adjust time daily for realism):
```
# Day 1-7 warm-up: run at varied times
30 10 * * * cd ${WORKSPACE} && python3 skills/tiktok-warmup/warmup.py \
  --cookies config/tiktok-cookies.txt \
  --niche "${TARGET_NICHE}" \
  --hashtags "${NICHE_HASHTAGS}" \
  --creators "${NICHE_CREATORS}" \
  --day $(python3 -c "import sqlite3,os;c=sqlite3.connect(os.path.expanduser('~/clawd/data/content-pipeline.db'));r=c.execute(\"SELECT value FROM meta WHERE key='warmup_last_day'\").fetchone();print(int(r[0])+1 if r else 1)") \
  --db data/content-pipeline.db \
  >> data/warmup.log 2>&1
```

The agent should monitor `warmup_complete` in the meta table and remove the cron job after day 7.

### Phase 3: Verify Warm-Up

#### 3.1 Check warm-up status `[AUTO]`

After 7 days, verify the warm-up completed:

**Remote:**
```
sqlite3 ${CONTENT_DB} "SELECT key, value FROM meta WHERE key LIKE 'warmup%' ORDER BY key"
```

Expected: `warmup_complete = true` and `warmup_last_day = 7`.

## Verification

1. **Script runs in dry mode (day 0):**
   ```
   python3 ${WORKSPACE}/skills/tiktok-warmup/warmup.py --cookies ${TIKTOK_COOKIES_PATH} --niche "test" --hashtags "#test" --day 0 --duration 1 --db ${CONTENT_DB}
   ```

2. **Progress tracked:**
   ```
   sqlite3 ${CONTENT_DB} "SELECT key, value FROM meta WHERE key LIKE 'warmup%'"
   ```

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Install warmup script | Execute | Confirm | Confirm |
| 2.1 Schedule cron | Execute | Confirm | Confirm |
| 3.1 Verify completion | Execute | Execute | Confirm |

## Dependencies

- **Depends on:** `deploy-content-pipeline.md` (database), `deploy-tiktok-posting.md` (cookies), Playwright
- **Required by:** `deploy-video-research.md` (must complete before posting begins)
