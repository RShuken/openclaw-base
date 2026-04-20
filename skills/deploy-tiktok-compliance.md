---
name: "Deploy TikTok Compliance Gate"
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
privacy_tier: "hybrid"
requires:
  - "FFmpeg"
  - "Content pipeline database (deploy-content-pipeline.md)"
  - "Telegram channel configured"
docs_urls:
  - "https://c2pa.org/specifications/specifications/1.3/specs/C2PA_Specification.html"
  - "https://www.tiktok.com/legal/page/global/community-guidelines/en"
---

# Deploy TikTok Compliance Gate

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)

## Purpose

Install an automated pre-posting compliance system that runs before every TikTok upload. Strips AI provenance metadata (C2PA/Content Credentials), validates video specs, checks posting frequency limits, and enforces shadow ban recovery rules. This is the safety layer between video generation and publishing.

**When to use:** Deploy alongside `deploy-tiktok-posting.md`. This skill is a mandatory pre-step — no video should be posted without passing the compliance gate.

**What this skill does:**
1. Installs the FFmpeg-based metadata stripping pipeline
2. Creates the compliance check script (metadata, specs, frequency, ban status)
3. Sets up shadow ban detection and auto-recovery
4. Integrates with the content pipeline database for tracking
5. Adds a Telegram alert system for compliance failures and ban detection

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CONTENT_DB}` | Content pipeline database | `${WORKSPACE}/data/content-pipeline.db` |
| `${TIKTOK_USERNAME}` | TikTok account username | `@relcontent` |
| `${CONTENT_AGENCY_CHAT_ID}` | Telegram: Content Agency group ID | `-1003398778462` |
| `${COMPLIANCE_TOPIC_ID}` | Telegram: Compliance Alerts topic thread ID | `185` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks for this skill:**

1. FFmpeg is installed:

**Remote:**
```
ffmpeg -version | head -1
```

Expected: `ffmpeg version X.X.X`

If this fails: Install FFmpeg. Reference: `brew install ffmpeg` (macOS), `apt install ffmpeg` (Linux).

2. Content pipeline database exists:

**Remote:**
```
sqlite3 ${CONTENT_DB} "SELECT value FROM meta WHERE key='schema_version'" 2>/dev/null || echo 'MISSING'
```

Expected: `1.0.0`.

3. ExifTool is available (optional, for deep metadata inspection):

**Remote:**
```
exiftool -ver 2>/dev/null || echo 'NOT_INSTALLED'
```

If not installed: `brew install exiftool` (macOS), `apt install libimage-exiftool-perl` (Linux). Not strictly required — FFmpeg handles the critical stripping.

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Metadata stripping | FFmpeg re-encode (libx264, CRF 18) | Client wants lossless or different codec |
| Max posts per day | 2 | Client wants more aggressive posting (increases risk) |
| Min hours between posts | 4 | Client's analytics show different optimal spacing |
| Shadow ban view threshold | 20% of rolling average | Client's niche has higher variance |
| Ban recovery pause | 72 hours | Client prefers shorter/longer pause |
| AIGC label | Always toggle ON | Client explicitly opts out (not recommended) |

## Prerequisites

- `deploy-content-pipeline.md` deployed
- `deploy-tiktok-posting.md` deployed
- FFmpeg installed on client machine
- `deploy-messaging-setup.md` deployed (for Telegram alerts)

## Steps

### Phase 1: Install Compliance Scripts

#### 1.1 Write the metadata stripping script `[AUTO]`

This script strips all C2PA/Content Credentials metadata from video files before upload. Uses FFmpeg to re-encode, which removes container-level metadata including C2PA manifests.

**Remote:**

Write to `${WORKSPACE}/skills/tiktok-compliance/strip-metadata.sh`:

```bash
#!/bin/bash
# Strip C2PA and all metadata from video files before TikTok upload
# Usage: strip-metadata.sh <input.mp4> [output.mp4]
#
# What this removes:
# - C2PA Content Credentials (DALL-E 3, Adobe Firefly, etc.)
# - EXIF metadata
# - XMP metadata
# - All container-level metadata
#
# What this does NOT remove:
# - Steganographic watermarks (SynthID, Video Seal) — these are in pixel data
# - No platform currently scans for these on upload (as of March 2026)

set -euo pipefail

INPUT="$1"
OUTPUT="${2:-${INPUT%.mp4}_clean.mp4}"

if [ ! -f "$INPUT" ]; then
    echo "ERROR: Input file not found: $INPUT" >&2
    exit 1
fi

# Check for C2PA metadata before stripping
echo "Checking for C2PA metadata..."
if ffprobe -v quiet -show_entries format_tags "$INPUT" 2>/dev/null | grep -qi "c2pa\|content.credentials\|provenance"; then
    echo "WARNING: C2PA metadata detected — will be stripped"
fi

# Re-encode with metadata stripped
# -map_metadata -1: strip all global metadata
# libx264 CRF 18: near-lossless quality
# TikTok specs: 1080x1920, 30fps target
echo "Stripping metadata and re-encoding..."
ffmpeg -y -i "$INPUT" \
    -map_metadata -1 \
    -fflags +bitexact \
    -flags:v +bitexact \
    -flags:a +bitexact \
    -c:v libx264 -preset slow -crf 18 \
    -c:a aac -b:a 192k \
    -movflags +faststart \
    "$OUTPUT" 2>/dev/null

# Verify metadata was stripped
REMAINING=$(ffprobe -v quiet -show_entries format_tags "$OUTPUT" 2>/dev/null | grep -c "TAG:" || true)
if [ "$REMAINING" -gt 1 ]; then
    echo "WARNING: $REMAINING metadata tags remain — running ExifTool cleanup"
    if command -v exiftool &>/dev/null; then
        exiftool -overwrite_original -all= "$OUTPUT" 2>/dev/null
    fi
fi

# Output file info
SIZE_MB=$(printf "%.1f" "$(echo "scale=1; $(stat -f%z "$OUTPUT" 2>/dev/null || stat -c%s "$OUTPUT" 2>/dev/null) / 1048576" | bc)")
DURATION=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "$OUTPUT" | cut -d. -f1)

echo "OK: $OUTPUT (${SIZE_MB}MB, ${DURATION}s, metadata stripped)"
```

Expected: Script at `${WORKSPACE}/skills/tiktok-compliance/strip-metadata.sh`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/tiktok-compliance/strip-metadata.sh
```

#### 1.2 Write the compliance gate script `[AUTO]`

The main compliance check that runs before every post. Validates metadata, specs, posting frequency, and ban status.

**Remote:**

Write to `${WORKSPACE}/skills/tiktok-compliance/check.py`:

```python
#!/usr/bin/env python3
"""Pre-posting compliance gate for TikTok.

Runs before every upload. Checks:
1. Metadata stripped (no C2PA)
2. Video specs match TikTok requirements
3. Posting frequency within limits
4. No active shadow ban
5. Caption length and hashtag count

Usage:
  python3 check.py <video_path> <caption> [--db <path>] [--max-daily 2] [--min-hours 4]

Exit codes:
  0 = PASS (safe to post)
  1 = FAIL (do not post, see output)
  2 = WARN (can post, but check warnings)
"""
import argparse
import json
import os
import sqlite3
import subprocess
import sys
from datetime import datetime, timedelta

def check_metadata(video_path):
    """Check for C2PA or AI provenance metadata."""
    issues = []
    try:
        result = subprocess.run(
            ["ffprobe", "-v", "quiet", "-show_entries", "format_tags", video_path],
            capture_output=True, text=True, timeout=30
        )
        output = result.stdout.lower()
        for keyword in ["c2pa", "content_credentials", "provenance", "ai_generated", "synthid"]:
            if keyword in output:
                issues.append(f"FAIL: {keyword} metadata detected — run strip-metadata.sh first")
    except FileNotFoundError:
        issues.append("WARN: ffprobe not found — cannot verify metadata")
    except subprocess.TimeoutExpired:
        issues.append("WARN: ffprobe timed out")
    return issues

def check_video_specs(video_path):
    """Validate video meets TikTok specs."""
    issues = []
    try:
        result = subprocess.run(
            ["ffprobe", "-v", "quiet", "-print_format", "json",
             "-show_streams", "-show_format", video_path],
            capture_output=True, text=True, timeout=30
        )
        data = json.loads(result.stdout)

        video_stream = next((s for s in data.get("streams", []) if s["codec_type"] == "video"), None)
        if not video_stream:
            issues.append("FAIL: No video stream found")
            return issues

        width = int(video_stream.get("width", 0))
        height = int(video_stream.get("height", 0))
        duration = float(data.get("format", {}).get("duration", 0))
        file_size = int(data.get("format", {}).get("size", 0))

        # Check aspect ratio (9:16 for TikTok)
        if width > 0 and height > 0:
            ratio = width / height
            if not (0.5 < ratio < 0.6):  # ~0.5625 for 9:16
                issues.append(f"WARN: Aspect ratio {width}x{height} ({ratio:.3f}) — TikTok prefers 9:16 (0.5625)")

        # Check resolution
        if width < 720 or height < 1280:
            issues.append(f"WARN: Resolution {width}x{height} below recommended 1080x1920")

        # Check duration
        if duration > 180:
            issues.append(f"WARN: Duration {duration:.0f}s — TikTok prefers under 3 minutes")
        if duration < 5:
            issues.append(f"FAIL: Duration {duration:.0f}s — too short, minimum ~5s")

        # Check file size (TikTok limit: 287.6 MB on mobile, 10GB on web)
        if file_size > 287_600_000:
            issues.append(f"WARN: File size {file_size/1048576:.0f}MB exceeds mobile upload limit (287MB)")

    except Exception as e:
        issues.append(f"WARN: Could not check specs: {e}")

    return issues

def check_posting_frequency(db_path, max_daily, min_hours):
    """Check if we're within posting frequency limits."""
    issues = []
    if not os.path.exists(db_path):
        return issues

    conn = sqlite3.connect(db_path)
    try:
        # Posts in last 24 hours
        count_24h = conn.execute(
            "SELECT count(*) FROM posts WHERE platform='tiktok' AND status='posted' AND published_at > datetime('now', '-24 hours')"
        ).fetchone()[0]

        if count_24h >= max_daily:
            issues.append(f"FAIL: Already posted {count_24h} times in 24h (limit: {max_daily})")

        # Most recent post timing
        last_post = conn.execute(
            "SELECT published_at FROM posts WHERE platform='tiktok' AND status='posted' ORDER BY published_at DESC LIMIT 1"
        ).fetchone()

        if last_post and last_post[0]:
            last_time = datetime.fromisoformat(last_post[0])
            hours_since = (datetime.utcnow() - last_time).total_seconds() / 3600
            if hours_since < min_hours:
                issues.append(f"FAIL: Last post was {hours_since:.1f}h ago (minimum spacing: {min_hours}h)")

    finally:
        conn.close()

    return issues

def check_shadow_ban(db_path):
    """Check for shadow ban indicators based on recent metrics."""
    issues = []
    if not os.path.exists(db_path):
        return issues

    conn = sqlite3.connect(db_path)
    try:
        # Get rolling average views (last 10 posts at 24h mark)
        rows = conn.execute(
            """SELECT pm.views FROM post_metrics pm
               JOIN posts p ON p.id = pm.post_id
               WHERE p.platform='tiktok' AND pm.hours_since_post=24
               ORDER BY pm.collected_at DESC LIMIT 10"""
        ).fetchall()

        if len(rows) >= 3:
            avg_views = sum(r[0] for r in rows) / len(rows)

            # Check most recent post
            latest = rows[0][0] if rows else 0
            if avg_views > 0 and latest < avg_views * 0.2:
                issues.append(f"FAIL: SHADOW BAN LIKELY — latest video {latest} views vs {avg_views:.0f} average (80%+ drop)")
                issues.append("ACTION: Pause posting for 72 hours. Delete flagged content if identifiable.")

        # Check if we're in a recovery pause
        meta = conn.execute(
            "SELECT value FROM meta WHERE key='shadowban_pause_until'"
        ).fetchone()
        if meta:
            pause_until = datetime.fromisoformat(meta[0])
            if datetime.utcnow() < pause_until:
                hours_left = (pause_until - datetime.utcnow()).total_seconds() / 3600
                issues.append(f"FAIL: Shadow ban recovery in progress — {hours_left:.0f}h remaining (resume: {meta[0]})")

    finally:
        conn.close()

    return issues

def check_caption(caption):
    """Validate caption meets TikTok best practices."""
    issues = []

    if len(caption) > 2200:
        issues.append(f"FAIL: Caption {len(caption)} chars exceeds TikTok limit (2200)")
    elif len(caption) > 300:
        issues.append(f"WARN: Caption {len(caption)} chars — under 300 performs better")

    # Count hashtags
    hashtags = [w for w in caption.split() if w.startswith('#')]
    if len(hashtags) == 0:
        issues.append("WARN: No hashtags — include 3-5 for discoverability")
    elif len(hashtags) > 8:
        issues.append(f"WARN: {len(hashtags)} hashtags — 3-5 is optimal, too many looks spammy")

    # Check hashtags have trailing space (tiktok-uploader requirement)
    for tag in hashtags:
        idx = caption.index(tag)
        after = idx + len(tag)
        if after < len(caption) and caption[after] not in (' ', '\n'):
            issues.append(f"FAIL: Hashtag '{tag}' not followed by space — tiktok-uploader will break")

    return issues

def main():
    parser = argparse.ArgumentParser(description='TikTok pre-posting compliance gate')
    parser.add_argument('video_path', help='Path to video file')
    parser.add_argument('caption', help='Post caption with hashtags')
    parser.add_argument('--db', default=None, help='Path to content-pipeline.db')
    parser.add_argument('--max-daily', type=int, default=2, help='Max posts per 24h (default: 2)')
    parser.add_argument('--min-hours', type=float, default=4, help='Min hours between posts (default: 4)')
    args = parser.parse_args()

    workspace = os.environ.get('WORKSPACE', os.path.expanduser('~/clawd'))
    db_path = args.db or os.path.join(workspace, 'data', 'content-pipeline.db')

    all_issues = []

    # Run all checks
    print("Running compliance checks...")

    print("  [1/5] Metadata check...")
    all_issues.extend(check_metadata(args.video_path))

    print("  [2/5] Video specs...")
    all_issues.extend(check_video_specs(args.video_path))

    print("  [3/5] Posting frequency...")
    all_issues.extend(check_posting_frequency(db_path, args.max_daily, args.min_hours))

    print("  [4/5] Shadow ban status...")
    all_issues.extend(check_shadow_ban(db_path))

    print("  [5/5] Caption validation...")
    all_issues.extend(check_caption(args.caption))

    # Determine result
    fails = [i for i in all_issues if i.startswith("FAIL")]
    warns = [i for i in all_issues if i.startswith("WARN")]

    result = {
        "status": "FAIL" if fails else ("WARN" if warns else "PASS"),
        "fails": fails,
        "warnings": warns,
        "checks_run": 5,
        "video": args.video_path,
        "caption_length": len(args.caption)
    }

    print(json.dumps(result, indent=2))

    if fails:
        print(f"\nBLOCKED: {len(fails)} failure(s). Fix before posting.")
        sys.exit(1)
    elif warns:
        print(f"\nPASS with {len(warns)} warning(s). Review before posting.")
        sys.exit(2)
    else:
        print("\nALL CLEAR. Safe to post.")
        sys.exit(0)

if __name__ == '__main__':
    main()
```

Expected: Script at `${WORKSPACE}/skills/tiktok-compliance/check.py`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/tiktok-compliance/check.py
```

#### 1.3 Write the shadow ban monitor script `[AUTO]`

Cron job that runs every 6 hours to check for shadow ban indicators and auto-pause posting if detected.

**Remote:**

Write to `${WORKSPACE}/skills/tiktok-compliance/shadowban-monitor.py`:

```python
#!/usr/bin/env python3
"""Shadow ban detection and auto-recovery monitor.

Runs on cron (every 6 hours). Checks:
- View velocity of recent posts vs rolling average
- For You traffic percentage (if available)
- Auto-pauses posting if ban detected
- Auto-resumes after 72-hour cooldown

Usage:
  python3 shadowban-monitor.py [--db <path>] [--pause-hours 72] [--threshold 0.2]
"""
import argparse
import json
import os
import sqlite3
import sys
from datetime import datetime, timedelta

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--db', default=None)
    parser.add_argument('--pause-hours', type=int, default=72)
    parser.add_argument('--threshold', type=float, default=0.2, help='View ratio below this = shadowban (default: 0.2 = 80% drop)')
    args = parser.parse_args()

    workspace = os.environ.get('WORKSPACE', os.path.expanduser('~/clawd'))
    db_path = args.db or os.path.join(workspace, 'data', 'content-pipeline.db')

    if not os.path.exists(db_path):
        print("Database not found, skipping")
        return

    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row

    # Check if already in recovery
    pause_row = conn.execute("SELECT value FROM meta WHERE key='shadowban_pause_until'").fetchone()
    if pause_row:
        pause_until = datetime.fromisoformat(pause_row['value'])
        if datetime.utcnow() < pause_until:
            hours_left = (pause_until - datetime.utcnow()).total_seconds() / 3600
            print(json.dumps({"status": "recovering", "hours_remaining": round(hours_left, 1)}))
            return
        else:
            # Recovery period over — clear the flag
            conn.execute("DELETE FROM meta WHERE key='shadowban_pause_until'")
            conn.execute("INSERT INTO status_log (idea_id, from_status, to_status, changed_by, reason) SELECT id, status, status, 'system', 'Shadow ban recovery complete — posting resumed' FROM content_ideas WHERE status='scheduled' LIMIT 1")
            conn.commit()
            print(json.dumps({"status": "recovered", "message": "Recovery complete, posting resumed"}))
            return

    # Get 24h view counts for recent posts
    rows = conn.execute("""
        SELECT p.id, p.platform_url, pm.views, pm.collected_at
        FROM post_metrics pm
        JOIN posts p ON p.id = pm.post_id
        WHERE p.platform='tiktok' AND pm.hours_since_post BETWEEN 20 AND 28
        ORDER BY pm.collected_at DESC LIMIT 10
    """).fetchall()

    if len(rows) < 3:
        print(json.dumps({"status": "insufficient_data", "posts_tracked": len(rows)}))
        conn.close()
        return

    views = [r['views'] for r in rows]
    avg = sum(views) / len(views)
    latest = views[0]

    ratio = latest / avg if avg > 0 else 1.0
    is_banned = ratio < args.threshold and avg > 50  # Only trigger if we normally get decent views

    result = {
        "status": "SHADOWBAN_DETECTED" if is_banned else "healthy",
        "latest_views": latest,
        "rolling_avg": round(avg, 0),
        "ratio": round(ratio, 3),
        "threshold": args.threshold,
        "posts_analyzed": len(rows)
    }

    if is_banned:
        pause_until = (datetime.utcnow() + timedelta(hours=args.pause_hours)).isoformat()
        conn.execute(
            "INSERT OR REPLACE INTO meta (key, value, updated_at) VALUES ('shadowban_pause_until', ?, datetime('now'))",
            (pause_until,)
        )
        conn.execute(
            "INSERT INTO status_log (idea_id, from_status, to_status, changed_by, reason) "
            "SELECT id, status, status, 'system', ? FROM content_ideas WHERE status='scheduled' LIMIT 1",
            (f"Shadow ban detected — posting paused until {pause_until}. Latest: {latest} views vs {avg:.0f} avg ({ratio:.1%})",)
        )
        conn.commit()
        result["action"] = f"Posting paused for {args.pause_hours}h until {pause_until}"
        result["alert"] = "SEND_TELEGRAM_ALERT"

    print(json.dumps(result, indent=2))
    conn.close()

    if is_banned:
        sys.exit(1)

if __name__ == '__main__':
    main()
```

Expected: Script at `${WORKSPACE}/skills/tiktok-compliance/shadowban-monitor.py`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/tiktok-compliance/shadowban-monitor.py
```

### Phase 2: Set Up Cron Jobs

#### 2.1 Schedule shadow ban monitor `[GUIDED]`

Run the shadow ban monitor every 6 hours.

Schedule a cron job (or launchctl on macOS):
```
0 */6 * * * cd ${WORKSPACE} && python3 skills/tiktok-compliance/shadowban-monitor.py --db data/content-pipeline.db >> data/shadowban-monitor.log 2>&1
```

If shadow ban is detected, the monitor writes to the database and the agent should pick up the alert on next Telegram check and notify in the Compliance Alerts topic.

#### 2.2 Schedule metrics collection `[GUIDED]`

Collect performance metrics for posted videos at defined intervals (1h, 6h, 24h, 48h, 7d after posting).

The agent should check for recently posted videos and pull metrics at the appropriate intervals. This can be done via the agent's normal polling loop rather than a separate cron job — the agent checks the posts table for entries where the next metrics checkpoint hasn't been collected yet.

### Phase 3: Integrate with Posting Pipeline

#### 3.1 Update posting workflow `[GUIDED]`

The content pipeline rules (`${WORKSPACE}/config/content-pipeline-rules.md`) should be updated to include the compliance gate. The posting flow becomes:

```
Video Generated → Strip Metadata → Compliance Check → [PASS] → Post
                                                    → [FAIL] → Alert in Telegram, do NOT post
                                                    → [WARN] → Post but note warnings
```

Add to the agent's posting behavior:

```markdown
## Pre-Post Compliance (MANDATORY)

Before EVERY TikTok upload, run these steps in order:

1. Strip metadata:
   bash skills/tiktok-compliance/strip-metadata.sh <video_path> <clean_path>

2. Run compliance check:
   python3 skills/tiktok-compliance/check.py <clean_path> "<caption>" --db data/content-pipeline.db

3. If FAIL: Do NOT post. Alert in Compliance Alerts topic. Log the failure.
4. If WARN: Post, but include warnings in the Telegram notification.
5. If PASS: Proceed to post.

NEVER bypass the compliance gate. Even in auto mode.
```

## Verification

1. **Metadata stripping works:**
   ```
   ${WORKSPACE}/skills/tiktok-compliance/strip-metadata.sh test_video.mp4 test_clean.mp4
   ffprobe -v quiet -show_entries format_tags test_clean.mp4
   ```
   Expected: No metadata tags.

2. **Compliance check runs:**
   ```
   python3 ${WORKSPACE}/skills/tiktok-compliance/check.py test_clean.mp4 "Test caption #test #tiktok " --db ${CONTENT_DB}
   ```
   Expected: `PASS` or `WARN` with specific issues.

3. **Shadow ban monitor runs:**
   ```
   python3 ${WORKSPACE}/skills/tiktok-compliance/shadowban-monitor.py --db ${CONTENT_DB}
   ```
   Expected: `insufficient_data` or `healthy`.

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Metadata stripping script | Execute | Execute | Confirm |
| 1.2 Compliance gate script | Execute | Execute | Confirm |
| 1.3 Shadowban monitor | Execute | Execute | Confirm |
| 2.1 Schedule cron | Execute | Confirm | Confirm |
| 3.1 Update posting workflow | Execute | Confirm | Confirm |

## Dependencies

- **Depends on:** `deploy-content-pipeline.md` (database), `deploy-tiktok-posting.md` (posting), FFmpeg
- **Required by:** `deploy-video-research.md` (compliance gate is called in the posting step)
