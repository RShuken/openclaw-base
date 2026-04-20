---
name: "Deploy YouTube Posting"
category: "integration"
subcategory: "social-posting"
third_party_service: "YouTube Data API v3"
auth_type: "oauth"
openclaw_version_required: "2026.4.15+"
version_last_verified: "2026-04-20"
last_checked: "2026-04-20"
maintenance_status: "active"
model_env_var: "n/a"
clawhub_alternative: "upload-post"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "Google Cloud Console OAuth 2.0 credentials"
  - "Python 3.9+ with google-api-python-client"
  - "Content pipeline database"
docs_urls:
  - "https://developers.google.com/youtube/v3/docs/videos/insert"
  - "https://developers.google.com/youtube/v3/guides/uploading_a_video"
---

# Deploy YouTube Posting

## Compatibility
- **OpenClaw Version**: 2026.4.15+ (bumped 2026-04-20)
- **Status**: READY
- **Blocked Commands**: (none)

## Purpose

Install YouTube video upload capabilities via the official YouTube Data API v3. Supports regular videos, Shorts (auto-detected from vertical format), and Made for Kids flagging. Uses OAuth2 with a stored refresh token — one-time consent, then fully automated forever.

**When to use:** When the client wants automated YouTube posting alongside TikTok. This is the most reliable upload method available — official API, no browser automation, no cookies to expire.

**What this skill does:**
1. Installs Google API Python packages
2. Guides one-time OAuth consent flow
3. Creates the upload script (videos, Shorts, Made for Kids)
4. Integrates with the content pipeline database
5. Tests with a private upload

## How Commands Are Sent

Standard protocol — see `_authoring/_deploy-common.md`. `Remote:` commands run on the target machine; `Operator:` commands run locally.

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${WORKSPACE}` | Client profile: workspace path | `~/clawd/` |
| `${CONTENT_DB}` | Content pipeline database | `${WORKSPACE}/data/content-pipeline.db` |
| `${YOUTUBE_CLIENT_SECRETS}` | Google Cloud Console → OAuth 2.0 credentials download | `${WORKSPACE}/config/youtube-client-secrets.json` |
| `${YOUTUBE_TOKEN}` | Generated during OAuth consent | `${WORKSPACE}/config/youtube-token.json` |
| `${YOUTUBE_CHANNEL_NAME}` | The YouTube channel name | `Capybear` |

## Before You Start

Follow the **Standard Deployment Protocol** in `_deploy-common.md`.

**Pre-flight checks:**

1. Python 3.9+ is available:
   ```
   python3 --version
   ```

2. Content pipeline database exists:
   ```
   sqlite3 ${CONTENT_DB} "SELECT value FROM meta WHERE key='schema_version'" 2>/dev/null
   ```

3. Client secrets file exists:
   ```
   test -f ${YOUTUBE_CLIENT_SECRETS} && echo 'EXISTS' || echo 'MISSING'
   ```
   If missing: the human needs to create OAuth credentials at console.cloud.google.com (see Phase 2).

## Adaptation Points

| Component | Default | Customize When... |
|-----------|---------|-------------------|
| Default privacy | `private` (safe default) | Client wants public by default |
| Category | `22` (People & Blogs) | Client's niche fits a different category |
| Made for Kids | `false` | Client is creating kids content (legal requirement) |
| AI disclosure | `true` (always disclose) | Client is not using AI generation |
| Upload quota | 6/day (default) | Apply for quota extension at Google |

## Prerequisites

- `deploy-content-pipeline.md` deployed
- Google Cloud project with YouTube Data API v3 enabled
- OAuth 2.0 client secrets file (`client_secrets.json`)
- One-time browser access for OAuth consent

## Steps

### Phase 1: Install Dependencies

#### 1.1 Install Google API packages `[AUTO]`

**Remote:**
```
python3 -m pip install google-api-python-client google-auth-oauthlib google-auth-httplib2
```

Expected: All packages installed successfully.

### Phase 2: OAuth Setup `[HUMAN_INPUT]`

#### 2.1 Guide human through Google Cloud setup

Tell the human:

> **Set up YouTube API access (10 minutes):**
>
> 1. Go to **https://console.cloud.google.com**
> 2. Click **Select a Project** → **New Project** → name it (e.g., "Content Pipeline") → Create
> 3. In the left sidebar: **APIs & Services** → **Library**
> 4. Search for **YouTube Data API v3** → click it → **Enable**
> 5. Go to **APIs & Services** → **Credentials**
> 6. Click **+ Create Credentials** → **OAuth client ID**
> 7. If prompted for consent screen: choose **External**, fill in app name + your email, save
> 8. Application type: **Desktop app** → name it → Create
> 9. Click **Download JSON** on the credentials you just created
> 10. Send me that JSON file

Wait for the human to provide the `client_secrets.json` file.

#### 2.2 Transfer client secrets to workspace `[HUMAN_INPUT]`

Transfer the downloaded JSON to `${YOUTUBE_CLIENT_SECRETS}` using base64 file transfer.

**Remote:**
```
mkdir -p ${WORKSPACE}/config
```

Then write the file content to `${YOUTUBE_CLIENT_SECRETS}`.

#### 2.3 Run one-time OAuth consent `[HUMAN_INPUT]`

Run the OAuth flow. This opens a browser window where the human clicks "Allow" once. After that, the refresh token is saved and no browser is ever needed again.

**Remote:**
```
python3 -c "
from google_auth_oauthlib.flow import InstalledAppFlow
SCOPES = ['https://www.googleapis.com/auth/youtube.upload', 'https://www.googleapis.com/auth/youtube']
flow = InstalledAppFlow.from_client_secrets_file('${YOUTUBE_CLIENT_SECRETS}', SCOPES)
creds = flow.run_local_server(port=0)
with open('${YOUTUBE_TOKEN}', 'w') as f:
    f.write(creds.to_json())
print('OAuth consent complete. Token saved.')
"
```

Expected: Browser opens, human clicks Allow, `youtube-token.json` is saved.

If this fails: Ensure the OAuth consent screen is configured. If "app not verified" warning appears, click "Advanced" → "Go to [app name] (unsafe)" → Allow. This is expected for personal projects.

### Phase 3: Create Upload Script

#### 3.1 Write the YouTube upload script `[AUTO]`

Create the upload script that handles regular videos, Shorts, and Made for Kids content.

**Remote:**

Write to `${WORKSPACE}/skills/youtube-post/upload.py`:

```python
#!/usr/bin/env python3
"""YouTube video upload via official Data API v3.

Usage:
  python3 upload.py <video_path> --title "Title" --description "Desc" \
    [--tags "tag1,tag2"] [--privacy public|private|unlisted] \
    [--made-for-kids] [--ai-content] [--schedule "2026-03-20T12:00:00Z"] \
    [--category 22] [--post-id <id>] [--db <path>] [--dry-run]

Shorts: upload a vertical video (9:16) under 3 minutes — auto-detected.
"""
import argparse
import json
import os
import sqlite3
import sys
from datetime import datetime

def get_youtube_service(client_secrets, token_path):
    from google.oauth2.credentials import Credentials
    from google.auth.transport.requests import Request
    from googleapiclient.discovery import build

    creds = Credentials.from_authorized_user_file(token_path)
    if creds.expired and creds.refresh_token:
        creds.refresh(Request())
        with open(token_path, 'w') as f:
            f.write(creds.to_json())
    return build("youtube", "v3", credentials=creds)

def update_post_status(db_path, post_id, status, platform_post_id=None, platform_url=None, error=None):
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
    parser = argparse.ArgumentParser(description='Upload video to YouTube')
    parser.add_argument('video_path', help='Path to video file')
    parser.add_argument('--title', required=True, help='Video title (max 100 chars)')
    parser.add_argument('--description', default='', help='Video description')
    parser.add_argument('--tags', default='', help='Comma-separated tags')
    parser.add_argument('--privacy', default='private', choices=['public', 'private', 'unlisted'])
    parser.add_argument('--made-for-kids', action='store_true', help='Flag as Made for Kids (COPPA)')
    parser.add_argument('--ai-content', action='store_true', default=True, help='Flag as AI/synthetic content (default: true)')
    parser.add_argument('--schedule', default=None, help='Publish time ISO 8601 (requires privacy=private)')
    parser.add_argument('--category', default='22', help='YouTube category ID (22=People & Blogs)')
    parser.add_argument('--post-id', type=int, default=None, help='Content pipeline post ID')
    parser.add_argument('--db', default=None, help='Content pipeline database path')
    parser.add_argument('--secrets', default=None, help='Path to client_secrets.json')
    parser.add_argument('--token', default=None, help='Path to youtube-token.json')
    parser.add_argument('--dry-run', action='store_true', help='Validate without uploading')
    args = parser.parse_args()

    workspace = os.environ.get('WORKSPACE', os.path.expanduser('~/clawd'))
    db_path = args.db or os.path.join(workspace, 'data', 'content-pipeline.db')
    secrets_path = args.secrets or os.path.join(workspace, 'config', 'youtube-client-secrets.json')
    token_path = args.token or os.path.join(workspace, 'config', 'youtube-token.json')

    if not os.path.exists(args.video_path):
        print(f"ERROR: Video not found: {args.video_path}", file=sys.stderr)
        sys.exit(1)

    tags = [t.strip() for t in args.tags.split(',') if t.strip()] if args.tags else []

    body = {
        "snippet": {
            "title": args.title[:100],
            "description": args.description,
            "tags": tags,
            "categoryId": args.category,
        },
        "status": {
            "privacyStatus": args.privacy,
            "selfDeclaredMadeForKids": args.made_for_kids,
        },
    }

    # AI content disclosure
    if args.ai_content:
        body["status"]["containsSyntheticMedia"] = True

    # Scheduled publishing
    if args.schedule:
        body["status"]["privacyStatus"] = "private"
        body["status"]["publishAt"] = args.schedule

    if args.dry_run:
        print(json.dumps({
            "status": "dry_run",
            "video": args.video_path,
            "body": body,
            "file_size_mb": round(os.path.getsize(args.video_path) / 1048576, 1)
        }, indent=2))
        return

    # Upload
    update_post_status(db_path, args.post_id, 'posting')

    try:
        from googleapiclient.http import MediaFileUpload

        youtube = get_youtube_service(secrets_path, token_path)
        media = MediaFileUpload(args.video_path, chunksize=-1, resumable=True)

        request = youtube.videos().insert(
            part="snippet,status",
            body=body,
            media_body=media
        )

        print("Uploading to YouTube...")
        response = None
        while response is None:
            status, response = request.next_chunk()
            if status:
                pct = int(status.progress() * 100)
                print(f"  {pct}% uploaded")

        video_id = response["id"]
        video_url = f"https://youtu.be/{video_id}"

        result = {
            "status": "posted",
            "video_id": video_id,
            "url": video_url,
            "privacy": args.privacy,
            "made_for_kids": args.made_for_kids,
            "ai_disclosed": args.ai_content
        }
        print(json.dumps(result, indent=2))

        update_post_status(db_path, args.post_id, 'posted',
                          platform_post_id=video_id, platform_url=video_url)

    except Exception as e:
        error_msg = str(e)
        print(json.dumps({"status": "failed", "error": error_msg}), file=sys.stderr)
        update_post_status(db_path, args.post_id, 'failed', error=error_msg)
        sys.exit(1)

if __name__ == '__main__':
    main()
```

Expected: Script at `${WORKSPACE}/skills/youtube-post/upload.py`.

**Remote:**
```
chmod +x ${WORKSPACE}/skills/youtube-post/upload.py
```

### Phase 4: Test Upload

#### 4.1 Dry-run test `[GUIDED]`

**Remote:**
```
python3 ${WORKSPACE}/skills/youtube-post/upload.py /dev/null \
  --title "Test" --description "Test" --dry-run
```

Expected: JSON output with dry-run status.

#### 4.2 Private upload test `[GUIDED]`

Upload a test video as PRIVATE (not visible to anyone).

**Remote:**
```
python3 ${WORKSPACE}/skills/youtube-post/upload.py "${WORKSPACE}/output/videos/test.mp4" \
  --title "Pipeline Test - Delete Me" \
  --description "Automated upload test" \
  --privacy private \
  --ai-content \
  --db ${CONTENT_DB}
```

Expected: Video uploaded as private, YouTube URL returned.

If this fails:
- Token expired → run OAuth consent again (Phase 2.3)
- Quota exceeded → wait until midnight Pacific, or request quota extension
- `NoLinkedYouTubeAccount` → the OAuth was done with an account that has no YouTube channel. Create a channel first at youtube.com.

## Verification

1. **Dependencies installed:**
   ```
   python3 -c "from googleapiclient.discovery import build; print('OK')"
   ```

2. **Token exists and is valid:**
   ```
   python3 -c "
   from google.oauth2.credentials import Credentials
   c = Credentials.from_authorized_user_file('${YOUTUBE_TOKEN}')
   print('VALID' if not c.expired else 'EXPIRED')
   "
   ```

3. **Upload script runs:**
   ```
   python3 ${WORKSPACE}/skills/youtube-post/upload.py --help
   ```

## API Quota Reference

| Operation | Cost (units) | Default daily (10,000 units) |
|-----------|-------------|------------------------------|
| Upload video | 1,600 | ~6/day |
| Update video | 50 | ~200/day |
| List videos | 1 | ~10,000/day |

For more than 6 uploads/day, apply for quota extension at Google Cloud Console.

## Autonomy Mode Behavior

| Step | Auto | Guided | Manual |
|------|------|--------|--------|
| 1.1 Install packages | Execute | Execute | Confirm |
| 2.1-2.3 OAuth setup | SKIP (needs human) | Guide | Guide |
| 3.1 Write upload script | Execute | Execute | Confirm |
| 4.1 Dry-run test | Execute | Confirm | Confirm |
| 4.2 Private upload test | SKIP | Confirm | Confirm |

## Dependencies

- **Depends on:** `deploy-content-pipeline.md` (database), Google Cloud project with YouTube Data API v3
- **Required by:** `deploy-video-research.md` (multi-platform posting)
